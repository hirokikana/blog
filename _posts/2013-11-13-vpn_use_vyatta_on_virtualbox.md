---
layout: post
title: VirtualBox上でVyattaを使ってVPN
date: 2013-11-13 01:09:39.000000000 +09:00
type: post
published: true
status: publish
categories:
- dev
tags:
- Vyatta
meta:
  _edit_last: '1'
  _thumbnail_id: '454'
  _wp_old_slug: virtualbox%e4%b8%8a%e3%81%a7vyatta%e3%82%92%e4%bd%bf%e3%81%a3%e3%81%a6vpn
author:
  login: admin
  email: hiroki@hirokikana.com
  display_name: hiroki.kana
  first_name: Hiroki
  last_name: Takayasu
---
[  
](http://blog.hirokikana.com/wp-content/uploads/2013/11/Untitled.jpg)VyattaはLinuxベースのソフトウェアルーターです。各種Linuxディストリビューションでもルーターとしての役割を持たせることは可能ですが、構築や設定が割とめんどくさいというのがあると思います。Vyattaは必要なコンポーネントはあらかじめ用意され、よくあるネットワーク機器のようなCLIなどがあり非常に設定しやすく、手軽にルーターを用意することができます。また、仮想化環境用のイメージなども用意され仮想化環境で手軽に利用することもできます。

今回はVyattaでL2TP over IPSec環境をVirtualBox上で実現する方法を紹介します。前回に引き続きVirtualBox上なのは、やはりまずは検証環境と言えば仮想化環境でやろうという私の考えというか、まあその辺です。

### はじめに

まず今回は実際に外部のネットワーク経由で接続するのではなく、VirtualBoxの内部ネットワークにVPN経由で繋ぐ環境を構築してみます。図にすると下記のような構成です。

[![Untitled]({{ site.baseurl }}/assets/Untitled-1024x403.jpg)](http://blog.hirokikana.com/wp-content/uploads/2013/11/Untitled.jpg)

クライアントはMac OS X 10.8 Mountain Lionに標準のVPNクライアントを利用します。

### Vyattaのインストール

まずはVyattaをインストールする仮想マシンを作成します。特に良いスペックが必要にもないので、メモリ512MB、ストレージは8GBに今回はしました。用途によっては必要なスペックが変わってくると思いますが、今回は検証をするだけなのでそれほど考えずに…。

ネットワークデバイスは今回は2つ用意します。1つ目は外側のネットワークに繋ぐため「ブリッジアダプター」として設定します。2つ目は外からは直接つなげないネットワークを構成するためにVirtualBoxのホストOS同士しか通信できない「内部ネットワーク」に設定します。

次に[Vayattaのダウンロードページ](http://www.vyatta.org/downloads "Vyatta download" )からVC6.6R1のLive CD ISOをダウンロードします。最初はLive CDで立ち上げたうえでHDDにインストールして利用します。

Live CD起動後にユーザー名vyatta、パスワードvyattaでログインし下記コマンドを実行します。
    
    
    $ install system

これでHDDへのVyattaのインストールが開始されますが、その際いくつか質問をされます。下記に簡単にその問答をまとめました。

  * imageをHDDにインストールしても良いか　→　Yes
  * パーティションの設定をどのように行うか　→　Auto
  * どのHDDにイメージをインストールするか　→　sda
  * sdaのデータを全部消すけど良いか　→　Yes
  * rootパーティションのサイズ　→　そのままEnter(最大容量まで使う)
  * configのディレクトリはどこにするか　→　そのままEnter(デフォルトのパス)
  * vyattaのパスワード(確認含めて計2回)　→　任意のパスワード
  * GRUBのインストール先　→　そのままEnter(sda)



以上の質問に答えるだけでHDDへのインストールは完了です。Live CDを取り出したうえで「reboot now」コマンドで再起動します。再起動後はユーザ名vyatta、パスワードは先ほど入力したパスワードでログインします。

まずは設定の練習としてホスト名の変更を行います。
    
    
    $ configure
    # set system host-name <<router-name>>
    # commit
    # save
    # exit
    $ reboot now

これで再起動後にホスト名が設定したものになっていれば完了です。Vyattaでは設定したらcommitで反映、saveでファイルに保存を必ず行うようにします。Ciscoのルーターなどのネットワーク機器の設定と似ていますね。

次に各インターフェイスにIPアドレスを設定します。今回はeth0に外側のネットワークのIPアドレス、eth1には内部ネットワーク用の任意のプライベートアドレスを割り当てます。
    
    
    $ configure
    # set interfaces ethernet eth0 address 192.168.100.30/24
    # set system gateway-address 192.168.100.1
    # set system name-server 192.168.100.1
    # set interfaces ethernet eth1 address 192.168.101.1/24
    # commit
    # save
    # show interfaces
    # exit

show interfacesは設定した内容が正しく反映されているか確認するために、設定を表示しています。またeth0に指定したアドレスにpingが通るか確認します。これで基本的な設定は完了です。

### VPNの設定

まずはIPsecの設定を行います。
    
    
    $ configure
    # edit vpn ipsec
    # set ipsec-interface interface eth0
    # set nat-traversal enable
    # set nat-networks allowed-network 0.0.0.0/0
    # exit

ここではIPsecをどのネットワークからでも接続できるようにしています(allowed-networkでの設定)。特定の場所からのみの接続の場合はここをきちんと設定しておくことをおすすめします。

次にL2TPの設定を行います。認証モードにいくつかの種類があるのですが、今回は最もセキュアではないけれども、設定が手軽な全員の共通パスワードとユーザーごとのパスワードで認証するタイプのものを利用します。
    
    
    $ configure
    # edit vpn l2tp remote-access
    # set ipsec-settings authentication mode pre-shared-secret
    # set ipsec-settings authentication pre-shared-secret <ユーザー全員の共通パスワード>
    # set authentication mode local
    # set authentication local-users username <ユーザー名> password <パスワード>
    # set outside-address 192.168.100.30
    # set client-ip-pool start 192.168.101.100
    # set client-ip-pool stop 192.168.101.110
    # commit 
    # save 
    # exit

これでVyatta側の設定は完了です。

### Macからの接続

Macから特に追加でクライアントなどをインストールすることなく今回構築したVPNに接続することができます。まずは「システム環境設定」の「ネットワーク」の項目を開き、左下の＋ボタンをクリックします。

[![vyatta_mac_network_add]({{ site.baseurl }}/assets/スクリーンショット-2013-11-11-1.09.541-300x266.png)](http://blog.hirokikana.com/wp-content/uploads/2013/11/スクリーンショット-2013-11-11-1.09.541.png)

表示されたダイアログでインターフェイスに「VPN」VPNタイプで「L2TP over IPSec」を選択し、サービス名に任意の文字列を入力し「作成」ボタンをクリックします。

[![スクリーンショット 2013-11-11 1.11.32]({{ site.baseurl }}/assets/スクリーンショット-2013-11-11-1.11.32-300x164.png)](http://blog.hirokikana.com/wp-content/uploads/2013/11/スクリーンショット-2013-11-11-1.11.32.png)

 

作成されたVPNの構成から「構成の追加」を選び任意の名前をつけて「作成」ボタンを押します。

[![スクリーンショット 2013-11-11 1.14.23]({{ site.baseurl }}/assets/スクリーンショット-2013-11-11-1.14.23-300x110.png)](http://blog.hirokikana.com/wp-content/uploads/2013/11/スクリーンショット-2013-11-11-1.14.23.png)

次にサーバーアドレスにeth0に割り当てたIPアドレス、アカウント名にL2TPの設定で作成したユーザー名を入力し、「認証設定」ボタンをクリックします。認証設定画面でユーザ認証のパスワードにユーザーのパスワード、コンピュータ認証の共有シークレットにユーザー全員の共通パスワードを入力し「OK」ボタンを押します。

[![スクリーンショット 2013-11-11 1.16.06]({{ site.baseurl }}/assets/スクリーンショット-2013-11-11-1.16.06-298x300.png)](http://blog.hirokikana.com/wp-content/uploads/2013/11/スクリーンショット-2013-11-11-1.16.06.png)

 

これで接続ボタンを押すことでVPNに接続され192.168.101.0/24のネットワークのIPアドレスが付与されます。このこの状態で192.168.101.1にpingし疎通を確認しましょう。これで本来はVirtualBoxのホスト同士でしかやりとりできないネットワークにVPNで接続できました。さらに動作確認をする場合にはVirtualBox上に内部ネットワークのNICを持ったゲストOSを立ち上げて確認するのが良いでしょう。

[![スクリーンショット 2013-11-11 1.18.26]({{ site.baseurl }}/assets/スクリーンショット-2013-11-11-1.18.26-300x266.png)](http://blog.hirokikana.com/wp-content/uploads/2013/11/スクリーンショット-2013-11-11-1.18.26.png)

### まとめ

非常に簡単にVPNの環境を作ることができました。この辺を一般的なLinuxディストリビューションでもできるとは思うんですが、この手軽さはVyattaの特長かと思います。またCLIがLinuxのものとは違いネットワーク機器に近い形になっているのも、この後ネットワーク機器で同じような設定をする際に役立つのではないかと思います。

今回はVirtualBox上だったので問題にならなかったですが、VPN接続を待ち受けるIPアドレス(今回のeth0に割り当てたアドレス)はIPSecの設定で指定する必要があります。固定IPじゃない場合は、これが問題となってきます。しかしながら、簡易的に利用するのであればそのとき割り当てられたIPアドレスを指定して、割り当てアドレスが変わったら手動で変更するという運用でも割と良いかなと思っていたりします。

参考サイト

  * [Vyatta 6.6でPCをルータにする　BLOG.KAWATASO.NET](http://blog.kawataso.net/2013/06/07/vyatta-6-6%E3%81%A7pc%E3%82%92%E3%83%AB%E3%83%BC%E3%82%BF%E3%81%AB%E3%81%99%E3%82%8B/)
  * [Vyatta で自宅ルータ構築- jedipunkz' blog](http://jedipunkz.github.io/blog/2012/04/28/vyattarouter/)
  * [Vyatta で L2TP over IPsec による VPN 構築 - jedipunkz' blog](http://jedipunkz.github.io/blog/2013/08/24/vyatta-l2tp-ipsec-vpn/)



 
