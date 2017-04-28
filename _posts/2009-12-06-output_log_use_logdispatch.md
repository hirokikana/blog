---
layout: post
title: Log::Dispatchを使ってログ出力
date: 2009-12-06 20:22:52.000000000 +09:00
type: post
published: true
status: publish
categories:
- dev
tags:
- Perl
meta:
  _edit_last: '1'
  _wp_old_slug: logdispatch%e3%82%92%e4%bd%bf%e3%81%a3%e3%81%a6%e3%83%ad%e3%82%b0%e5%87%ba%e5%8a%9b
author:
  login: admin
  email: hiroki@hirokikana.com
  display_name: hiroki.kana
  first_name: Hiroki
  last_name: Takayasu
---
どんなアプリでもログ出力は必ずあるわけで。  
というわけで、Log::Dispatchを使用してログ出力をしてみたいと思います。

まず、ログの設定ファイルを下記のように書きます。
    
    
    
    dispatchers = file screen
    file.class = Log::Dispatch::File
    file.min_level = debug
    file.filename = /tmp/hogehoge             
    file.mode = append           
    file.format = [%d] [%p] %m at %F line %L %n
    
    screen.class = Log::Dispatch::Screen
    screen.min_level = debug
    screen.stderr = 1
    screen.format = [%d] [%p] %m %n
    

dispatchersには使用するLogger(もとい出力先)を指定します。  
今回はfileとscreenを作成しました。  
<クラス>.classで指定しているのは出力先ごとのクラスを指定します。  
min_levelには出力する最低限のログ、今回だと両方debugログ以上(つまりすべて)を出力するようにしています。  
infoなどとするとinfoログ以上(debugログ以外)が出力されます。

これを使用するスクリプトは以下のようにします。
    
    
    
    #!/usr/bin/perl   
    use warnings;
    use strict;
    use Log::Dispatch::Config;
    use Data::Dumper;
    
    Log::Dispatch::Config->configure('./log.conf');
    my $dispatcher = Log::Dispatch::Config->instance();
    
    $dispatcher->debug('デバッグログ');
    $dispatcher->info('インフォ');
    $dispatcher->info('インフォ');
    $dispatcher->info('インフォ');
                     
    my $hoge = {hoge => 'aaaa'};
    $dispatcher->warning(Dumper($hoge));                                           
    $dispatcher->error('error errro');
    $dispatcher->warning('warn warn');
    

ちなみに、screenにLog::Dispatch::Screenではなくて Log::Dispatch::Screen::Colorを指定するとログの種類によって色がつくようになります。

[  
![Log::Dispatch::Screen]({{ site.baseurl }}/assets/hogehoge-300x217.jpg)  
](http://blog.hirokikana.com/wp-content/uploads/2009/12/hogehoge.jpg)
