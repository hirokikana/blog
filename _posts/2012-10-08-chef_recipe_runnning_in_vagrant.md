---
layout: post
title: Vagrantを使ったChef recipeの動作確認環境の構築
date: 2012-10-08 23:35:04.000000000 +09:00
type: post
published: true
status: publish
categories:
- dev
tags:
- Chef
meta:
  _edit_last: '1'
  _thumbnail_id: '396'
  _wp_old_slug: vagrant%e3%82%92%e4%bd%bf%e3%81%a3%e3%81%9fchef-recipe%e3%81%ae%e5%8b%95%e4%bd%9c%e7%a2%ba%e8%aa%8d%e7%92%b0%e5%a2%83%e3%81%ae%e6%a7%8b%e7%af%89
author:
  login: admin
  email: hiroki@hirokikana.com
  display_name: hiroki.kana
  first_name: Hiroki
  last_name: Takayasu
---
何かアプリケーションを開発したりしている中で様々なミドルウェアをインストールしたり設定を更新したりすると思います。  
そんなインストールするパッケージのリストや設定はみなさんどのように管理していますか。  
私個人の場合はブログに書いておいたり、適当なメモ帳に書いたりしています。

そんなことをしていると起こるのが、もう1つ同じ環境を作ろうとしたときの作業がめんどくさいということですね。  
「ただホストが違うだけなのになぜこんなことに時間を取られにゃならんのだ」と思いますよね。

そんなときにSoftware Design 2012/10号にchefの特集を目にしました。  
Chefはrecipeという必要なパッケージや設定などを定義したファイルを元に環境を構築してくれるソフトウェアです。  
最初からChefを使って環境を作れば、同じ環境が必要になったときはChefのrecipeを実行するだけ。  
非常に便利そうですね。さっそく参考にさせていただき使ってみました。

recipeを書いていて問題になるのが、recipeが正しいことをどうやって確認すべきかということです。  
何事もまず「どうやって確認をするか」を考えること、はい、大切ですね。  
「○○すればできるけど確認が難しい」というものは、自分の首をジワジワと締め付け、最終的にはそういうものに締め上げられます。  
このようなマシンが丸ごと必要になるようなときは、仮想マシンの登場です。  
仮想マシンを利用してChefのrecipeが正しく動作するかを確認できれば、recipe書きもはかどりますね。

そこで今回はvagrantというVirtualBoxの仮想マシンの構築・設定をコマンドラインから管理することができるツールを使い、chefのrecipeの動作確認を行う環境の構築を行ってみようと思います。

### vagrantを使えるようにする

まず、VirtualBoxをインストールします。  
公式サイトの<https://www.virtualbox.org/>からダウンロードしてきてセットアップ。特に難しいことはないと思います。

次にgemを使ってvagrantをインストールします。
    
    
    
     $ sudo gem install vagrant
    

私の好みとしてはOSで元々入っていたものを汚したくないので、Pythonで言うところのpipやvirtualenv的なもので行えば良いのですが、今回は特に気にしないことにしました。

これでvagrantのセットアップは完了したので、早速使ってみます。  
<http://www.vagrantbox.es/>にvagrant用の各種OSのイメージがあるので、これを利用するのが手軽です。  
ただ、OSのイメージをダウンロードしてくるので回線細いと、そこそこ待ちます。  
今回はCentOS6.3の64bit版を利用してみます。
    
    
    
     $ vagrant box add centos https://dl.dropbox.com/u/7225008/Vagrant/CentOS-6.3-x86_64-minimal.box
     $ vagrant init centos
     $ vagrant up centos
    

これで仮想マシンが起動します。従来通りインストールするより断然早いですね。

このサーバーにsshでログインしたいときは下記のようにすることでvagrantユーザーでログインすることができます。
    
    
    
     $ vagrant ssh centos
    

この仮想マシンをシャットダウンする場合には下記コマンドを実行すると停止します。
    
    
    
     $ vagrant halt centos
    

もう一度起動する場合は再度upを行うことで起動することができます。

この仮想マシンを初期状態、つまり一番最初にupをしたときに戻す場合は一度destroyを行って再度upすることで初期状態に戻すことができます。
    
    
    
     $ vagrant destroy centos
     $ vagrant up centos
    

スナップショットと似ていますが、あくまでこれはboxという単位で管理されている別のものです。  
例えば同じ状態からキャッシュサーバーとDBサーバーを構築するとして、初期状態は同じでもその先インストールするものが違う場合、boxで管理されているvagrantを使えば簡単に複製することができます。  
ちょうどクラスとインスタンスの関係に似ていますね。  
これでvagrantを使うことができるようになりました。

### Chefのレシピを書く

次にChefの簡単なrecipeを書いてみたいと思います。  
今回は簡単にntpのインストールと設定を行うrecipeを作ろうと思います。

まずはrecipeのひな形を作成するコマンドknifeを利用したいので、chefをインストールします。
    
    
    
     $ sudo gem install chef --no-rdoc --no-ri
    

これでインストールが完了です。knifeコマンドが利用できるか確認しましょう
    
    
    
     $ which knife
     /usr/bin/knife
    

次に複数のrecipeを保存する場所として~/cookbooksを作成します
    
    
    
     $ mkdir ~/cookbooks
    

今後recipeはこのディレクトリ以下に作成します。

ここまでできたらrecipeのひな形を作成します。  
今回はntpのrecipeを作成するので、recipeの名前はntpにします。
    
    
    
     $ knife cookbook create ntp -o ~/cookbooks
    

このコマンドを実行することで~/cookbooks以下にntpというフォルダが作成されています。  
これがcookbookの中身です。

recipeの中で基本的な動作を定義するのが、ntp/recipes/default.rbです。  
ここにこのrecipeを実行した際にどのような動きをするかを定義します。  
今回はntpをインストール・自動起動、設定ファイルをあらかじめ用意したものに変更し再起動するというのを定義してみたいと思います。  
適当のエディタで~/cookbooks/ntp/recipes/default.rbを開き、下記のように編集してください。  
(#からはじまる行はコメントなので書かなくても動作には問題ありません)
    
    
    
     # ntpパッケージをインストール
     package "ntp" do
      action :install
     end
    
     # ntpdサービスを自動起動にし、このrecipe実行時に再起動する
     service "ntpd" do
      supports :status => true, :restart => true
      action [ :enable, :start ]
     end
    
     # あらかじめ用意した設定ファイルを/etc/ntp.confとして配置する
     template "/etc/ntp.conf" do
      source "ntp.conf"
      group "root"
      owner "root"
      mode "400"
      notifies :restart, "service[ntpd]"
     end
    

この中で出てきたtemplateのntp.confですが、~/cookbooks/ntp/templates/default/ntp.confとして下記のような内容を保存しておいてください。

このファイルがrecipeを実行したサーバーの/etc/ntp.confとして配置されます。
    
    
    
     driftfile /var/lib/ntp/ntp.drift
     statistics loopstats peerstats clockstats
     filegen loopstats file loopstats type day enable
     filegen peerstats file peerstats type day enable
     filegen clockstats file clockstats type day enable
    
     server ntp.nict.jp
     server ntp1.sakura.ad.jp
     server ntp.nc.u-tokyo.ac.jp
    
     restrict -4 default kod notrap nomodify nopeer noquery
     restrict -6 default kod notrap nomodify nopeer noquery
     restrict 127.0.0.1
     restrict ::1
    

このtemplateでは変数などを定義し、実行するサーバーやディストリビューションによって動作を変えることもでき、ある程度複雑な記述をすることができます。  
ひとまず、今回は特にそのあたりの機能は利用しません。

以上でntpサーバーを設定するrecipeを用意することができました。  
さて、次は実際にこのrecipeが正しく動作するのかというのをテストするためにvagrantで作成した仮想マシンでチェックしてみたいと思います。

### chefのレシピをvagrantを使ってチェックする

まずは、recipeを動作させるvagrantのbaseを決めたいと思います。  
ここでは先ほど使ったcentosを利用します。  
先ほどvagrant init centosを実行した際にVagrantfileが作成されたと思います。  
これはvagrantの起動する仮想マシンに関する設定ファイルです。

まずはVagrantfileの中身を整理したいと思います。  
今回はchef_testという識別名でcentsのboxを利用した仮想マシンを動作させたいと思うので、Vagrantfileを下記のように編集してください。
    
    
    
     # -*- mode: ruby -*-
     # vi: set ft=ruby :
    
     Vagrant::Config.run do |config|
      config.vm.define :chef_test do |c|
        c.vm.box = "centos"
        c.vm.box_url = "https://dl.dropbox.com/u/7225008/Vagrant/CentOS-6.3-x86_64-minimal.box"
        c.vm.network :hostonly, "192.168.33.10"
        c.vm.customize do |v|
          v.memory_size = 512
          v.name = "chef_test"
        end
        c.vm.provision :chef_solo do |prov|
          prov.cookbooks_path = "~/cookbooks"
          provad_recipe "ntp"
        end
      end
     end
    

下部にprovisionと書かれた部分にchef_soloの設定があります。  
これがchefを動作させるrecipeを定義する場所です。  
本来は仮想マシン側に単独実行させるためのchef soloやchefコマンドに渡すjsonなどを作成しなければいけないのですが、vagrantを利用すれば、自動的に実行してくれます。  
また、chef soloは配布されているイメージを利用すればあらかじめインストールされています。

さて、この状態でchef_testを起動します。
    
    
    
     $ vagrant up chef_test
    

起動時に緑色の文字で下記のような出力があったら指定したrecipeが実行されています。

実際にntpが設定されたかはログインし確認します。

これでrecipeを実行できる環境ができました。  
初期状態に戻したい場合はvagrant destory chef_testを実行することでboxの初期状態に戻すことができます。

毎回up/destroyを繰り返すのも起動時の待ち時間などがもったいなかったり、とりあえずrecipeやtemplateを修正して実行したい場合などは下記コマンドを実行することで、起動している仮想マシンへ対してchefのrecipe実行のみを行うこともできます。
    
    
    
     $ vagrant provision chef_test
    

### まとめ

これでvagrantを利用してChefのrecipeの動作確認が手軽にできるようになったと思います。  
vagrantは配布されているイメージだけではなく、自分の好きなboxを用意することも可能ということなので、それぞれの環境に合わせて利用するboxを用意することもできそうです。  
vagrantで手元でサーバーの設定を確認してから実マシンに適用とするようにすれば安心して実マシンへrecipeを適用することができるようになるでしょう。

ちなみに、WindowsやLinuxでもたぶん動作すると思いますが、MacOS 10.8 Mountain Lionのみで確認を行っています。
