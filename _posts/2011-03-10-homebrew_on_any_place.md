---
layout: post
title: homebrewを任意の場所で運用する
date: 2011-03-10 23:25:43.000000000 +09:00
type: post
published: true
status: publish
categories:
- dev
tags:
- homebrew
- Mac
meta:
  _edit_last: '1'
  _wp_old_slug: homebrew%e3%82%92%e4%bb%bb%e6%84%8f%e3%81%ae%e5%a0%b4%e6%89%80%e3%81%a7%e9%81%8b%e7%94%a8%e3%81%99%e3%82%8b
author:
  login: admin
  email: hiroki@hirokikana.com
  display_name: hiroki.kana
  first_name: Hiroki
  last_name: Takayasu
---
かれこれ前回の投稿から年をまたいでしまいました。  
ついつい間が空いてしまいます。

最近までMacではMacPortを使ってパッケージ管理を行っていました。  
しかし、ここんとこMacPortsに代わってhomebrewというのがあるというのを聞くようになったので、ちょっと使ってみることにしました。

homebrewはMac上で動くパッケージ管理を行うソフトウェアでMacPortsと違ってOSに既に入っているパッケージを最大限利用します。  
そのため、MacPortではOSに既に入っているものも依存関係がはられているものはかたっぱしからインストールしてしまいインストールに時間がかかってましたが、その辺がhomebrewを使えば導入にかかる時間を短縮することができます。

homebrewは/usr/local/以下がデフォルトのprefixに指定されており、ここにパッケージがすべて導入されます。  
また、pprefixで指定したところ以外を汚染することはないそうです。

特に/usr/local/以下で問題ないという方は下記のようにするだけで簡単に導入することができます。
    
    
    
    % ruby -e "$(curl -fsS http://gist.github.com/raw/323731/install_homebrew.rb)"
    

あとはbrewコマンドを使ってごにょごにょ…という話なのですが、その辺はひとまず置いといて今回はこのprefixを変更して任意の場所で運用したいと思います。  
なぜ任意の場所である必要があるかというと、複数のMacで同期させて使いたいので、Dropbox上のフォルダ以下で運用できれば幸せだなぁというのと、/usr/local/以下に導入する場合にroot権限が必要なのでなんとなく変なところ触ってるんじゃないかという根拠のない不安があるからです。

さて、任意の場所で運用するのはとても簡単です。  
今回は例として~/Dropbox/myhomebrew/以下で運用してみることにします。  
下記コマンドを実行するだけで、~/Dropbox/myhomebrew/以下にhomebrew一式がインストールされます。
    
    
    
    mkdir ~/Dropbox/myhomebrew/
    curl -Lsf http://github.com/mxcl/homebrew/tarball/master | tar xz --strip 1 -C ~/Dropbox/myhomebrew/
    

上記はgithubからtarballをダウンロードしてきて任意のディレクトリに展開をしています。  
homebrewは任意のディレクトリに展開をして、そこにインストールされたbrewコマンドを使用すれば、このディレクトリ以下ですべて運用されます。  
あとは、~/Dropbox/myhomebrew/bin/にパスを通しておきます。

試しに、実際本当にこのディレクトリ以下で運用できているかwgetコマンドをhomebrew経由でインストールしてみます。
    
    
    
    ~/Dropbox/myhomebrew/bin/brew install wget
    

実際にwgetコマンドが~/Dropbox/myhomebrew/bin/以下にインストールされています。
    
    
    
    iMac% ll ~/Dropbox/myhomebrew/bin
    total 40
    -rwxr-xr-x  1 hiroki  staff  12647  3 10 15:36 brew
    lrwxr-xr-x  1 hiroki  staff     28  3 10 23:17 wget -> ../Cellar/wget/1.12/bin/wget
    

と、ここまで書いて自分のMacBookで同期されたDropboxのwgetをたたいてみたところpermission deniedが…。  
どうやらDropboxはパーミッションまでは同期してくれない(?)模様です。

まぁ、同期はいくらでも方法はあるはずなので別の方法を模索してみます。  
というかそもそも他のPCでコンパイルしたものがデフォルトの状態で動くのだろうか…。  
課題はありますが、今回はひとまず別のディレクトリでhomebrewを運用するというところまでで、同期についてはまたうまくいったら書きたいと思います。  
# homebrewの使い方はbrewコマンド実行すれば出たり、参考サイトを参照していただければ…。

**参考サイト**

  * [homebrew Installation - GitHub](https://github.com/mxcl/homebrew/wiki/Installation)
  * [Macのパッケージ管理をMacPortsからhomebrewへ - よんちゅBlog](http://d.hatena.ne.jp/yonchu/20110226/1298723822)
  * [mac ports やめました! ー homebrew で快適 OSX 生活! - TokuLog 改メ tokuhirom’s blog](http://d.hatena.ne.jp/tokuhirom/20100625/1277435268)


