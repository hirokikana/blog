---
layout: post
title: CPANからスクリプトで自動的にRPMを作成する
date: 2009-11-16 01:15:45.000000000 +09:00
type: post
published: true
status: publish
categories:
- dev
tags:
- Perl
meta:
  _edit_last: '1'
  _wp_old_slug: cpan%e3%81%8b%e3%82%89%e3%82%b9%e3%82%af%e3%83%aa%e3%83%97%e3%83%88%e3%81%a7%e8%87%aa%e5%8b%95%e7%9a%84%e3%81%abrpm%e3%82%92%e4%bd%9c%e6%88%90%e3%81%99%e3%82%8b
author:
  login: admin
  email: hiroki@hirokikana.com
  display_name: hiroki.kana
  first_name: Hiroki
  last_name: Takayasu
---
Perlのモジュールを使用するときは、CPANよりインストールを行うと思います。  
しかし、CPANだとあとから何をインストールしたかや同じ環境を作る際に若干面倒です。  
そこで、CPANで公開されているモジュールをRPM化したいと思います。

手動でやるなど様々な方法があるようですが、今回は[こちら](http://d.hatena.ne.jp/stanaka/20090219/1234997257)で公開されているスクリプトを使用させてもらうことにしました。  
まず、スクリプトの実行に必要なモジュールなどをインストールしました。
    
    
    
    # cpan -i CPANPLUS
    # cpan -i RPM::Specfile
    # cpan -i YAML
    # yum install rpm-build
    

まっさらな環境に導入したのでいろいろ依存があることに気づきました。

次に参考サイトの通りにスクリプト類をgitで落としてきて使用します。
    
    
    
    git clone git://github.com/stanaka/cpan-dependency.git
    cd cpan-dependency
    perl ./bin/cpan-dependency.pl --conf=config/conf.yml Linux::Smaps
    

処理が終了するとカレントディレクトリに依存するモジュールを含めたrpmとspecファイルが生成されています。

configファイルに依存関係に関する情報などを記述することもできるようなので、CPANにあがってるモジュールできちんと依存関係が書いていないモジュールも安心だそうです。

**参考サイト**

  * [とあるはてな社員の日記 - CPANモジュールをスクリプト一発で依存解決しつつrpm化する](http://d.hatena.ne.jp/stanaka/20090219/1234997257)


