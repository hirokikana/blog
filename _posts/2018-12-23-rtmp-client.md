---
layout: post
title: RTMPの理解を深めようとしたが深まらなかった話
date: 2018/12/23 00:00:00 +09:00
type: post
published: true
status: publish
categories:
- dev
tags:
- RTMP
---
この記事は [ドワンゴ Advent Calendar 2018](http://qiita.com/advent-calendar/2018/dwango)の23日目です。

### はじめに
去年の記事があまりに雑であると反省し、書いた直後は「もう枠を無駄に使ってしまうのは申し訳ないな、来年からはやめよう」という気持ちでいました。
故に今年も枠を使うことに迷いがありました。しかし、私のようなものが雑な記事を書くことで他のメンバーの記事が引き立つのであればそれは全然関係無い時期にただ書くよりは意味があるのではないかという考えに至り今年も懲りずに記事を書くことにしました。まとまりの無い長文になってしまいましたが、お時間のある方は最後までお付き合いください。

さて、近年ではWebRTCといった新たなライブストリーミングを実現にあたる様々な解決策の選択肢が増えています。しかしながらRTMP(Real Time Message Protocol)で配信を行い、HLS(HTTP Live Streaming)等に変換し視聴するというスタイルは現在ではまだ広く利用されるライブストリーミングの方法であると考えられます。
今回はまだ広く利用されるストリーミングプロトコルであるRTMPについて理解を深めることを目的としてRTMPのクライアントを作成し、その際にぶつかったこと等を紹介します。

### 目的と今回のゴール
今回まずはRTMPプロトコルへの理解を深めるということを目的としました。このような記事でよくある困ってることを解決したいというようなキレイな目的はありません。
理解を深めるのであれば仕様を丁寧に読み込めばそれは理解したと言えるかもしれませんが、私の理解力はそれほど高くないためもう少し手を動かす必要があります。そのためには実現したいことをでっち上げて何らかのゴールを作る必要がありました。

ともあれAdobeから公開されている[RTMPの仕様書](https://www.adobe.com/jp/devnet/rtmp.html)を読みながら考えていこうということにしました。その読む段階でRTMPで送ることができるのは映像・音声だけでなく任意のデータも送ることができるということを知りました。これは映像と音声のデータに合わせて文字列のデータを送り字幕やコメントとして利用できるのではないかと考えました。

### 概要
RTMPサーバーをnginx-rtmpモジュールを利用したものとし、RTMPのクライアント(配信および視聴用)を実装するという方針としました。最終的に映像・音声を表示しながらコメントを表示するというのはさすがに間に合わないだろうというところから、まずは任意のデータを送受信できるところまでは実施しました。

### RTMP通信のシーケンス
RTMP通信は下記のようなシーケンスで通信が行われます。
 - Handshake
 - connect
 - createStream
 - 配信を行う場合
   - publish
 - 視聴を行う場合
   - play

#### Handshake
まずはクライアントからC0とC1というメッセージをサーバに送ります。C0は1byteで利用するRTMPバージョンが入り、現在では0〜2が非推奨となり3が格納されます。C1は1536byteで4byteのタイムスタンプ、4byteの0で埋められたメッセージ、1528byteのランダムな値が格納されています。ランダムな値は暗号的に安全である必要は無いとのことで今回は `random.getrandbits` で生成した値を利用しました。C0を受け取ったサーバはS0とS1を返します。S0とC0、S1とC1と内容のフォーマットは同一です。

その後クライアントからC2サーバーからS2を返してHandshakeが完了となります。C2/S2は4byteのタイムスタンプとパケットを読み取ったタイムスタンプ、続いてそれぞれS1/C2に含まれたランダムな値が1528byte格納されて返されます。今回利用したnginx-rtmpを利用したRTMPサーバーの場合にはC0+C1を送るとS0+S1+S2が返却され、クライアントからC2を送るという流れになっていました。C2を送っても何かHandshakeが終わった等のメッセージはありません。

![handshake](/img/post/2018-12-23_handshake_capture.png)

#### connect
Handshakeが終わったら次にNetConnection Commandsの `connect` メソッドを利用してどのRTMPサーバーのアプリケーションに接続します。
よくあるRTMPは以下のような形になっており、概ねパスの最初に指定されているのがアプリケーションです。

HandshakeまではRTMPとしては特殊なフォーマットをしていたのですが、ここから先にはチャックというフォーマットで送られます。
チャンクはチャンクヘッダとチャンクデータにわかれそれぞれ下記のようなものが含まれています。
 - チャンクヘッダ
   - Basicヘッダ(1byte~3byte) : チャンクを識別するチャンクストリームIDとメッセージヘッダのフォーマットを示す識別子
   - メッセージヘッダ(0, 3, 7, 11byte) : チャンクデータの種類(映像や音声なのかコマンドなのか等)やチャンクデータの長さ等の情報
   - Extend Timestamp(0, 4byte) : タイムスタンプが16777215を超えた際に利用される
 - チャンクデータ
 
それぞれWiresharkで見るとRTMP Header / RTMP Bodyという形でわかりやすく表示されています。

![wireshark_rtmp](/img/post/2018-12-23_wireshark_rtmp.png)

`connect` は接続に必要なデータがチャンクに含まれています。メッセージヘッダ内にあるチャンクデータ種を表すType IDは本来20(AMF0 Command Message)になりそうですがffmpegから行った際にはここが14になっていました。詳細な理由を調べれば何か歴史的経緯があったりしたかもしれないのですが、ここはひとまず動いているやりとりにしてしまった方が良いだろうと14を利用しました。

チャンクデータにはひとまず下記のものだけを含めましたが、正しく利用できました。仕様書にはこれ以外にも多くの指定することが可能な項目がありますが必要な際に付与すれば良いと思います。`connect` メソッドのチャンクデータAMF(Action Message Format)0というAction Scriptで利用されることを想定したシリアライズフォーマットが利用されます。AMF0の仕様はこちらの[PDF](https://www.adobe.com/content/dam/acom/en/devnet/pdf/amf0-file-format-specification.pdf) で公開されています。
  - コマンド名(`connect`)
  - トランザクションID(`connect` の場合は1)
  - コマンドオブジェクト
    - app : アプリケーション名
    - tcUrl : サーバーのURL
    - flashVer : 接続するFlash Playerのバージョン
    - オブジェクト終了のマーカー

nginx-rtmpモジュールにおいては `connect`メソッドが正しくサーバ側で受け取ると下記のようなメッセージがサーバから返ってきます。
  - Windows Acknowlegement Size
  - Set Peer Bandwidth
  - Set Chunk Size

#### createStream
次にNetConnection Commandsの `createStream` メソッドを利用し実際に流す論理的なストリームを生成します。
チャンクデータは下記のもので構成されています。

  - コマンド名(`createStream`)
  - トランザクションID
  - コマンドオブジェクトもしくはnull
  
トランザクションIDとコマンドオブジェクトに何を指定するのが正しいのかいまいち判断がつかなかったのでffmpegで配信した際のキャプチャした内容で、トランザクションIDが4、コマンドオブジェクトはnullが指定されていたのでひとまず今回はそれを指定しました。

#### publish
配信する際にはNetStream Commandsというストリームを操作するコマンドの `publish` メソッドを利用します。
  - コマンド名(`publish`)
  - トランザクションID(`publish` の場合は0)
  - null
  - ストリーム名
  - publish種別(ライブストリーミングの場合は`live`を指定)
送信後 `onStatus` コマンドで`NetStream.Publish.Start`が返されたら成功しており、配信を開始することができるようになります。

#### play
一方、視聴を行う場合はNetStream Commandの `play` メソッドを利用します。
  - コマンド名(`play`)
  - トランザクションID(`play` の場合は0)
  - null
  - ストリーム名
  - スタート時間(ストリーミングの際はおそらく0?)
`publish` と同じく `onStatus` コマンドで `NetStream.Play.Start`が返されたら成功しており、接続したストリームに配信が行われていたら映像・音声等のデータがサーバから送信されてきます。

### 便利なテクニック

#### Wiresharkを使ったデバッグ
Wiresharkはパケットキャプチャを行うソフトウェアです。同じ役割でCLI上で利用する際はtcpdumpをよく利用します。RTMPに限らずネットワーク上のデータを確認する際にはサーバー側でtcpdumpを走らせ、クライアント側でWiresharkを走らせるというのは原因の切り分けを行ううえでは非常に有効な手段となります。

仕様書を見て間違いが無く実装できれば全く問題無いのですが、多くの場合は間違っていたりあとは仕様書では具体的な値がわからなかったりという部分が出てきます。
今回はffmpegを使って配信して内容をWiresharkで確認しそれを正しい値として採用することにしました。そこになぜその値をきちんと理解していない部分も正直多くあったのですが、まずはそれっぽく動かしたいという気持ちが先行しました。

Wiresharkは便利なことにRTMPの通信のみをフィルタに`rtmpt`と入力することで絞り込むことができます。さらにチャンクヘッダとチャンクデータはParseされ見やすい形に表示されます。WiresharkでParseができない場合はTCPの通信として表示されるのでPayloadのバイナリを参照して正しい挙動の際との差分を確認します。

![wireshark-filter](/img/post/2018-12-23_wireshark_filter.png)

#### Python3でのバイナリの表現
Wiresharkやtcpdumpでキャプチャした際によくわからないけどバイナリで見ると差分がある際にとりあえず意味はわからずとも同内容のバイナリを送りたいということがありました。例えば `70 6c 61 79` というバイナリを作りたいときには下記のようにします。
```
>>> (0x70).to_bytes(1,'big') + (0x6c).to_bytes(1, 'big') + (0x61).to_bytes(1,'big') + (0x79).to_bytes(1,'big')
b'play'
```
上記の結果からわかるように`70 6c 61 79`は`play`という文字列をバイナリで表記したものです。
とりあえず試してみたいという部分は上記の方法で無理やり同じバイナリを送って動作確認をしました。

このような方法はPython3から実現可能になっており、Pythonでバイナリ操作をするならPython2ではなくPython3を利用するのが望ましいです。


### 作成したもの
[https://github.com/hirokikana/rtmp-client](https://github.com/hirokikana/rtmp-client)

`bin/rtmp-client` コマンドでは`publish`と`play`の両方を行うことができます。
まず`play`で待ち受けて`test message`という文字列を`publish`するのは下記のようなコマンドで実施します。

```
$ python3 bin/rtmp-client play
```

```
$ echo "test message" | python3 bin/rtmp-client publish
```

このようにすることで`play`で待機しているターミナルで`test message`と表示されるでしょう。

![demo](/img/post/2018-12-23_run_demo.gif)

### 課題
#### Video / Audio 以外のデータをクライアントに直接送ることはできない
仕様書を見ると任意のデータを配信者からサーバーを介して複数の視聴者に送ることができるように見えました。しかし実際行ってみるとRTMPサーバーから先へ任意のデータを送ることができませんでした。
nginx-rtmpをきちんとデバッグしたわけではないのですが[このあたり](https://github.com/arut/nginx-rtmp-module/blob/master/ngx_rtmp_live_module.c#L1126-L1130)から察するというか、雰囲気としてもしかしてVideo / Audio以外のメッセージは素通ししないようにしているのではないかと考えています。

RTMPのプロトコル上では任意データを送信することができるが、サーバから視聴者へのデータを送るかどうかについてはサーバの実装によるということのようでした。これに気づいた時には目論見がはずれでっち上げたゴールには締切までにたどり着けないなという気持ちになりました。今回とりあえず任意のデータを送るためにVideo / Audioのメッセージであるように見せかけて送るようにしました。

#### 受信した内容が送信した内容と一致しない
当初小さなJPEG画像をpublishしてテストをしたのですが、受信側で正しく展開されませんでした。受信したバイナリを確認してみるといくつかのサイズからバイナリが異なっていました。Wiresharkで確認してみるとその段階では送信したバイナリと一致している様子だったのでPythonで受信した際に何か余計なことになってしまっているのではないかということまではわかりました。

これに気づいたのがつい最近ということもあり、クライアントだけ新しく作って検証するのも時間が無いうえ、rtmpdump等のクライアントでも内容を確認できなかったので今回は課題として残る形になりました。

### まとめ
当初の目的であるRTMPへの理解を深めるというのは、最初から相対的に見ると深まったとはいえゴールとしていた成果にたどり着けなかったところからすると目的を達せたとは言えないでしょう。

しかしながら今回
  - Wireshark / tcpdump はとても便利
  - Pythonでバイナリ操作をするならPython3
  - RTMPでは任意データも視聴者に簡単に送れるわけではない
ということがわかったのは良かったという気持ちではいます。

### 参考リンク
 - [RTMP 1.0 準拠のサーバーをGo言語で実装する](https://developers.cyberagent.co.jp/blog/archives/13739/#flow-after-receiving-command-message)
 - [Adobe’s Real Time Messaging Protoco](http://wwwimages.adobe.com/www.adobe.com/content/dam/acom/en/devnet/rtmp/pdf/rtmp_specification_1.0.pdf)
 - [arut/nginx-rtmp-module](https://github.com/arut/nginx-rtmp-module/)
 - [Real-Time Messaging Protocol - Wikipedia](https://en.wikipedia.org/wiki/Real-Time_Messaging_Protocol)
 - [Action Message Format -- AMF0](https://www.adobe.com/content/dam/acom/en/devnet/pdf/amf0-file-format-specification.pdf)
 - [Action Message Format - Wikipedia](https://en.wikipedia.org/wiki/Action_Message_Format)
