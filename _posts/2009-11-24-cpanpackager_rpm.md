---
layout: post
title: CPAN::Packagerを使用してCPANモジュールをRPM化する
date: 2009-11-24 00:44:49.000000000 +09:00
type: post
published: true
status: publish
categories:
- dev
tags: []
meta:
  _edit_last: '1'
  _wp_old_slug: cpanpackager%e3%82%92%e4%bd%bf%e7%94%a8%e3%81%97%e3%81%a6cpan%e3%83%a2%e3%82%b8%e3%83%a5%e3%83%bc%e3%83%ab%e3%82%92rpm%e5%8c%96%e3%81%99%e3%82%8b
author:
  login: admin
  email: hiroki@hirokikana.com
  display_name: hiroki.kana
  first_name: Hiroki
  last_name: Takayasu
---
先日、cpan-dependencyスクリプトを使用したRPMの作成方法を記事にしたのですが、YAPC::Asia 2009でCPAN::Packagerというcpan-dependencyを参考にさらにパワーアップしたものが発表されていたのを知りました。  
cpan-dependencyでは依存関係関連やMakefile.PLではなくてBuild.PLを使用している場合などなど、いろいろ導入するのは大変でちょっと挫折してました。  
今回、CPAN::Packagerを使用したところ比較的導入しやすかったです。

参考サイトに従い、インストールを行いました。  
まず最初にrpm-buildパッケージに含まれる/usr/lib/rpm/perl.reqを修正します。  
これを修正しないとモジュール内でuseされているモジュールに関してRequireが張られてしまいます。  
実はこれで2、3日はまってました。
    
    
    
          # there is no easy way to find out if a file named systeminfo.ph
          # will be included with the name sys/systeminfo.ph so only use the
          # basename of *.ph files
    
          ($module  =~ m/\.ph$/) && next;
    
          $require{$module}=$version;
          $line{$module}=$_;
    

**修正前**
    
    
    
          # there is no easy way to find out if a file named systeminfo.ph
          # will be included with the name sys/systeminfo.ph so only use the
          # basename of *.ph files
    
          ($module  =~ m/\.ph$/) && next;
    
          #$require{$module}=$version;
          #$line{$module}=$_;
    

**修正後**

あとはCPAN::Packagerをインストールします。  
gitのリポジトリもありますが、これ自体が依存しているモジュールが多いのでCPANからインストールするのが楽です。
    
    
    
     # cpan -i CPAN::Packager
    

configファイルはgitからとってきます。
    
    
    
     # git clone git://github.com/dann/p5-cpan-packager.git
     # cd p5-cpan-packager
    

今後、conf/config-rpm.yamlを使用します。  
きっと随時updateがかかると思うので、gitの管理下に入れておくのがベターだと思います。  
最後に、設定ファイルのリポジトリをどこかのCPANミラーに設定するかmini CPANに設定します。

あとは、下記コマンドを実行すれば~/.cpanpackager以下にrpmが作成されます。
    
    
    
     # cpan-packager --module DBIx::Class --builder RPM --conf ~/p5-cpan-packager/conf/config-rpm.yaml
    

DBIx::ClassとCatalyst::Develのrpmを作成しましたが、Catalystは少しほかの依存モジュールを先にrpm化しないとダメだったりしましたが基本的にはうまくいきました。  
これでrpm化も手軽にできるようになりそうです。

  * [dann@webdev CPAN::PackagerでのRPM作成とインストール](http://dann.g.hatena.ne.jp/dann/20090401/p1)


