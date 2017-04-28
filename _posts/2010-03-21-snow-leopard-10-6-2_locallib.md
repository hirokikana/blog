---
layout: post
title: Snow Leopard 10.6.2でlocal::libを使う
date: 2010-03-21 02:35:33.000000000 +09:00
type: post
published: true
status: publish
categories:
- dev
tags:
- Perl
meta:
  _edit_last: '1'
  _wp_old_slug: snow-leopard-10-6-2%e3%81%a7locallib%e3%82%92%e4%bd%bf%e3%81%a3%e3%81%a6%e3%81%bf%e3%82%8b
author:
  login: admin
  email: hiroki@hirokikana.com
  display_name: hiroki.kana
  first_name: Hiroki
  last_name: Takayasu
---
いろいろなPerlモジュールが必要になったときはCPANを使用してインストールを行うと思います。  
そんなときにroot権限を持っていない場合やシステムのディレクトリを汚したくない場合などはいろいろと不便になります。  
解決策としては、先の記事でも書きましたが、パッケージ管理でRPMを使用できる場合はrpmを作成するのが良いと思います。

とはいえ、クライアントマシンでMacを使用しているのでRPMを使用できず、MacPortsを使用していました。  
MacPortsにないモジュールや最新のバージョンを使用したいときなどはやはり不便でした。  
そこで、任意のディレクトリにPerlモジュール群をインストールして使用できるlocal:libを使用してみることにしました。  
ほとんど参考サイトどおりですが、Mac OS X Snow Leopard 10.6.2環境下で導入してます。

まずは、local-libをダウンロードします。
    
    
    
    % curl -L -O http://search.cpan.org/CPAN/authors/id/A/AP/APEIRON/local-lib-1.005001.tar.gz
    % tar xzvf local-lib-1.005001.tar.gz
    % cd local-lib-1.005001
    

local::libとは関係ないですが、curlコマンドでリダイレクト先まで追ってダウンロードするときには-Lオプションが必要です。  
恥ずかしながらここで10分くらいはまりました…orz

次にMakefileつくってテストしてインストールします。  
ここではホームディレクトリ以下のperllocalディレクトリに導入しました。
    
    
    
    % perl Makefile.PL --bootstrap=$HOME/perllocal
    % make && make test
    % make install
    

最後に必要な環境変数を自動的に設定されるように下記を.zshrcに追加します。  
bashやcshの場合は適切な場所に同じことをしてください。
    
    
    
    % echo 'eval $(perl -I$HOME/perllocal/lib/perl5 -Mlocal::lib=$HOME/perllocal)' >> ~/.zshrc
    % source ~/.zshrc
    

これで導入は完了です。  
あとはcpanコマンドを使用してPerlモジュールをインストールするとperllocal以下に必要なモジュールがインストールされるはずです。  
さっそくAnyEventをインストールして確認してみます。
    
    
    
    % cpan -i AnyEvent
    % perldoc -l AnyEvent
    /Users/hiroki/perllocal/lib/perl5/AnyEvent.pm 
    

きちんとperllocal以下に導入されています。

コマンドラインから使用するのは上記の環境変数を設定しているので特に問題なく使用できますが、CGIなどの場合はスクリプトでuse libでライブラリのパスを指定する必要があります。  
例えば、下記のように指定します。
    
    
    
    #!/usr/bin/perl
    use strict;
    use lib ('/Users/hiroki/perllocal/lib/perl5');
    use local::lib('/Users/hiroki/perllocal/lib/perl5');
    use AnyEvent;
    
    my $cv_timer = AnyEvent->condvar;
    my $timer; $timer = AnyEvent->timer(
        after => 0,
        interval => 1,
        cb => sub {
            print AnyEvent->time, "\n";
            $cv_timer->send;
        },
    );
    $cv_timer->recv;
    

さらに、perllocalをSubversionなどで共有すればインストール作業は1回ですみそうな気がするのでいずれやってみます。  
同じアーキテクチャ同士じゃないとダメそうですが。

**参考サイト**

  * [local::libを使った非rootでのCPAN環境構築 - hide-k.net#blog](http://blog.hide-k.net/archives/2009/02/locallibrootcpa.php)
  * [XREA/CORESERVERに於けるlocal::libを使ったCPAN環境の構築 - Eorzea Lounge](http://blog.eorzea.asia/2009/07/post_40.html)


