---
layout: post
title: GALAXY NEXUS(4.1 JellyBean)でのセルスタンバイ問題
date: 2012-08-01 10:22:56.000000000 +09:00
type: post
published: true
status: publish
categories:
- dev
tags:
- android
meta:
  _edit_last: '1'
  _thumbnail_id: '368'
  _wp_old_slug: galaxy-nexus4-1-jellybean%e3%81%a7%e3%82%bb%e3%83%ab%e3%82%b9%e3%82%bf%e3%83%b3%e3%83%90%e3%82%a4%e5%95%8f%e9%a1%8c
author:
  login: admin
  email: hiroki@hirokikana.com
  display_name: hiroki.kana
  first_name: Hiroki
  last_name: Takayasu
---
さて、前回の投稿からずいぶん間が空き季節が変わってしまいました。

最近FactoryImageが公開されたGALAXY NEXUS用のJellyBeanのイメージを、手持ちのSC-04Dに入れました。  
「そんなに変わらないだろう」と思っていたのですが、入れてみたらヌルサク具合にもうICSには戻れなくなってしまいました。  
私はそんなGALAXY NEXUS(SC-04D)にIIJmioのSIMを入れて使っているのですが、このようなデータ専用SIMを入れると発生する問題としてセルスタンバイ問題というのがあります。  
今日はそれをうまく解決できたときの話をしようと思います。

## セルスタンバイ問題とは

よくある携帯キャリアで契約をした場合、音声通信とデータ通信が両方使用できるSIMをもらうことができます。  
しかしながら、日本通信(b-mobile)などのデータ通信のみしか使用できないSIMの場合、音声通信の回線が使用できないようになっており端末側で音声通信が圏外と認識されてしまい、アンテナピクトイメージが圏外の状態を表示します。

この状態でも正しくAPNの設定を行えばデータ通信を行うことは可能です。  
しかし圏外の状態の場合、携帯端末は電波をより頻繁に探すためバッテリの消耗がより激しくなります。  
それがセルスタンバイ問題と呼ばれる現象です。  
実際に端末「設定→電池」からもその様子を見ることができます。  
(写真を撮り忘れました…)

ICSを使っている頃からこの現象はあったのですが、特に電池の減りが早いという印象もなかったため放っておいたのですが、バッテリが余分に使われていると思うと何かもったいないので対策をしてみようと思ったというところです。

## セルスタンバイ問題の解決方法

解決策は、音声回線との圏内・圏外の判定をしないようにし、データ通信が可能ならば圏内とするようにすることで圏外の状態にならないようにします。

そのための方法を検索をするとみなさん同じことで悩んでいて、様々なサイトがヒットしました。  
特に[こちら](http://bl.oov.ch/ "こちら" )ではスクリプトが用意を作っておられたようで「これで簡単に…!」と思ったらWindowsのバッチファイルでした。

しかし特にWindowsに特化した処理があるわけでもありませんでしたので、Macで動くようにシェルスクリプトを作成しました。  
[fix_cellstandby.sh](https://github.com/hirokikana/tools/blob/master/android/fix_cellstandby.sh "https://github.com/hirokikana/tools/blob/master/android/fix_cellstandby.sh" )

このコマンドをたたけば必要なプログラムをダウンロードしてきたうえで端末にセルスタンバイ問題を解決するパッチを当てることができます。

### 動作環境

  * JellyBeanのFactoryImageが適用されて何もいじっていないGALAXY NEXUS(SC-04D)であること
  * adbで端末と接続できる状態であること
  * Clockwork mod recoveryなどで起動し、/system,/dataを自由にマウントできる状態にできること



adbでの接続環境を整えるあたりやClockworkで起動する手順などについては長くなるので省略します。

### 利用にあたり注意

この手順を行ったことによるデータの消失や端末の破損、文鎮化などが発生した場合でも、一切責任はおいません。**自己責任**で実行をお願いします。

### スクリプト動作手順

まず、上記リンクからfix_cellstandby.shをダウンロードし、ダウンロードしたディレクトリで下記を実行します。  
そのとき端末とはadb経由で接続できる状態にしておいてください。
    
    
     $ sh fix_cellstandby.sh

実行すると必要なプログラムをダウンロード、パッチの適用などを行ってくれます。  
しかし、これだけではダメで最後は/system内のファイルを書き換える必要があります。  
そのため、端末をClockworkなどで立ち上げ/systemと/data領域をマウントしたうえで、下記コマンドを実行します。  
(すいません、横に長いですが1行です)
    
    
     $ adb shell "cp /data/local/tmp/framework.odex /system/framework/framework.odex.new && mv /system/framework/framework.odex /system/framework/framework.odex.original && mv /system/framework/framework.odex.new /system/framework/framework.odex && sync && reboot"

これで端末が起動し、アンテナピクトが圏内を表示できれば成功です。

[![]({{ site.baseurl }}/assets/IMG_0100-300x225.jpg)](http://blog.hirokikana.com/dev/galaxy-nexus4-1-jellybean-cell-standby/attachment/img_0100/)

もし、起動しなくなったら/system/framework/framework.odex.originalを戻すか、FactoryImageを適用しなおしてください。

### 最後に

やはりアンテナピクトがあるとちゃんと動いている感があって気持ちがいいものですね。  
バッテリの持ちがどうなったかというのはまだ長く使っていないのでわかりません。  
何か明らかに変わった！とかがあったらまたここに追記しようと思います。

ちなみにこの方法だと音声通信の部分の圏内・圏外判定をごまかしているので音声も使えるSIMを差し込んだときに思いもよらないエラーが発生するかもしれません。  
なので、音声通信も行う場合は純な状態のFactoryImageを使用する方が良いでしょう。

また、このスクリプトはMacOS 10.8 Mountain LionとGALAXY NEXUS(SC-04D)でしか動作確認をしていません。  
「他の端末でも動いたよ！」とか言うのがあったらぜひ連絡をいただけるとうれしいです。

## 参考サイト

  * [ブローヴちゃん: Android + データ専用 SIM での動作修正パッチ](http://bl.oov.ch/2012/01/android-sim.html)
  * [nanamikuのブログ: IIJmio高速モバイル & Galaxy Nexusのセルスタンバイ問題を解決](http://nanamiku.blogspot.jp/2012/03/iijmio-galaxy-nexus.html "nanamikuのブログ: IIJmio高速モバイル & Galaxy Nexusのセルスタンバイ問題を解決" )


