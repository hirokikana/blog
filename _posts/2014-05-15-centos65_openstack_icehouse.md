---
layout: post
title: CentOS6.5にOpenStack(Icehouse)環境を構築
date: 2014-05-15 02:34:17.000000000 +09:00
type: post
published: true
status: publish
categories:
- dev
tags:
- OpenStack
meta:
  _edit_last: '1'
  _oembed_0bb491f56ccf3c7caa29350a23a00880: "{{unknown}}"
  _oembed_8a7605b284065ca092f6c6fe4518f1f2: "{{unknown}}"
  _oembed_f379daba3899682240152cbf88b35f8f: "{{unknown}}"
  _oembed_1bcfc51ba133a2ccd5ba0a9db2e8506d: "{{unknown}}"
  _wp_old_slug: centos6-5%e3%81%abopenstackicehouse%e7%92%b0%e5%a2%83%e3%82%92%e6%a7%8b%e7%af%89
author:
  login: admin
  email: hiroki@hirokikana.com
  display_name: hiroki.kana
  first_name: Hiroki
  last_name: Takayasu
---
先日、OpenStackの最新版であるIcehouseがリリースされました。RedHat系のOSにOpenStackの環境構築をする際にはよく利用するRDOのOpenStackリポジトリも早い段階でIcehouseになりました。今回はそのIcehouseをCentOS6.5に構築してみます。

前回の記事ではFedora18で構築しましたが、今回はCentOS6.5+packstackを利用して構築してみます。

## 仮想マシンの設定

物理マシンを用意するのは敷居が高いので、今回もVirtualBox上で構築をします。VirtualBoxはNested Virtualizationに対応していないのでqemuを利用して検証します。

ネットワーク構成はeth0にホストオンリーネットワークを割当し、インターネットに接続するためにeth1はNATを割当ます。eth0はOpenStack内でのpublicネットワークとし、FlaotingIPを割り当てます。そのためeth0はプロミスキャスモードを「すべて許可」にします。

[![setting_promiscuous_mode]({{ site.baseurl }}/assets/screenshot-2014-05-13-19.42.34-300x250.png)](http://blog.hirokikana.com/wp-content/uploads/2014/05/screenshot-2014-05-13-19.42.34.png)

この状態でCentOS6.5を最小構成でインストールしておき、eth0にはstatic IPアドレスを設定します。設定するIPアドレスはVirtualBoxのホストオンリーネットワークの設定に従います。Macの場合VirtualBox→環境設定→ネットワークのホストオンリーネットワークのタブ内から上記で設定したホストオンリネットワークの設定を確認し、そのネットワークで疎通するIPアドレスを決めます。

[![hostonly_netwrok]({{ site.baseurl }}/assets/screenshot-2014-05-13-19.46.54-300x196.png)](http://blog.hirokikana.com/wp-content/uploads/2014/05/screenshot-2014-05-13-19.46.54.png)

 

例えば上記の画像の用に表示されていた場合は192.168.33.0/24のネットワークで構成されているので、192.168.33.254などを設定するとよいでしょう。

OpenStackのインストール

OpenStackは前回同様packstackを利用してインストールします。前回はすでに用意されていたsetup.shを利用しましたが、今回は手動でインストールの定義を行うanswerfileを編集して行います。

まずはRDOとpackstackをインストールします。
    
    
    $ sudo yum -y update
    $ sudo yum install -y http://rdo.fedorapeople.org/rdo-release.rpm
    $ sudo yum install -y openstack-packstack

packstackのインストールが終わったらanswerfileのひな形を生成します。
    
    
    $ packstack --gen-answer-file=answer.conf

これでカレントディレクトリにanswer.confが生成されます。設定内容は推奨される状態となっているので、編集を行います。まず、設定されているIPアドレスがeth1(NAT)の方のものになってしまっているので、このままではホストOSからアクセスすることができません。IPアドレスはすべてホストオンリーネットワーク側のNIC(eth0)のIPアドレスに変更します。下記コマンドではeth0が192.168.33.254、eth1が10.0.3.15だった場合に置換を行います。
    
    
    $ sed -i -e 's/10.0.3.15/192.168.33.254/g' answer.conf

次にサンプルで作成されるテナントやネットワークを今回は特にいらないのでCONFIG_PROVISION_DEMOをnにします。これでanswerファイルの準備は完了です。

answerファイルができたらpackstackを実行します。
    
    
    $ packstack --answer-file=answer.conf

最初にrootパスワードを聞かれるので入力します。あとはひたすら待っていれば環境が構築されます。大体1時間程度かかるのでゆっくり待ちます。「 Installation completed successfully」と表示されたらインストール終了です。

最後にVirtualBoxではNestedVirtualizationが利用できないので/etc/nova/nova.confのlibvirt_type=kvmをlibvirt_type=qemuに変更しておきます。

http://&lt;eth0のIPアドレス&gt;/dashboard/にアクセスし、ログイン画面が表示されたらユーザー名admin、パスワードは/root/keystone_admin内のOS_PASSWORDの内容を入力しログインボタンを押します。Horizonのトップページが表示されればインストールは完了しています。

[![horizon_top]({{ site.baseurl }}/assets/screenshot-2014-05-13-21.06.09-300x164.png)](http://blog.hirokikana.com/wp-content/uploads/2014/05/screenshot-2014-05-13-21.06.09.png)

## OpenStackでのネットワーク設定

OpenStackで利用するネットワークはneutronを利用して管理されています。neutronはpluginで様々なネットワークコンポーネントを利用できるようになっていますがpackstackで特に指定しない場合はOpen vSwitchが利用される形で構成されます。これらの設定を行わないとOpenStackからネットワークを利用することができません。

まず最初に/etc/neutron/plugin.iniを開き、下記の項目を新たに追加します。追加する箇所は追加する項目についてコメントで書いてある下あたりに書くとわかりやすいでしょう。
    
    
    network_vlan_ranges = physnet1
    bridge_mappings = physnet1:br-ex

次に新たにできるネットワークデバイスのbr-exとeth0の設定を行います。eth0に192.168.33.254が割り当てられているとします。まずは/etc/sysconfig/network-scripts/ifcfg-br-exを下記内容で作成します。
    
    
    DEVICE=br-ex
    TYPE=OVSBridge
    DEVICETYPE=ovs
    ONBOOT=yes
    BOOTPROTO=static
    IPADDR=192.168.56.254
    NETMASK=255.255.255.0

次に既存の/etc/sysconfig/network-scripts/ifcfg-eth0を下記のように変更します。
    
    
    DEVICE=eth0
    HWADDR=<デバイスのMACアドレス。元から記載のもの>
    TYPE=OVSPort
    UUID=<元から記載のもの>
    ONBOOT=yes
    DEVICETYPE=ovs
    OVS_BRIDGE=br-ex

これでサーバーを再起動します。再起動後にifconfigコマンドで確認すると新たにbr-ex / br-intといったデバイスができていることが確認できると思います。上記の設定ではbr-exとeth0をブリッジする設定をしたのですが、下記のようにbr-exにIPアドレスが割り当てられeth0には割り当てられていないことをまず確認します。
    
    
    # ifconfig eth0
    eth0 Link encap:Ethernet HWaddr 08:00:27:2B:59:BB 
     inet6 addr: fe80::a00:27ff:fe2b:59bb/64 Scope:Link
     UP BROADCAST RUNNING MULTICAST MTU:1500 Metric:1
     RX packets:122 errors:0 dropped:0 overruns:0 frame:0
     TX packets:83 errors:0 dropped:0 overruns:0 carrier:0
     collisions:0 txqueuelen:1000 
     RX bytes:13396 (13.0 KiB) TX bytes:13879 (13.5 KiB)
    
    # ifconfig br-ex
    br-ex Link encap:Ethernet HWaddr 08:00:27:2B:59:BB 
     inet addr:192.168.56.254 Bcast:192.168.56.255 Mask:255.255.255.0
     inet6 addr: fe80::70d6:32ff:fed8:548c/64 Scope:Link
     UP BROADCAST RUNNING MTU:1500 Metric:1
     RX packets:148 errors:0 dropped:0 overruns:0 frame:0
     TX packets:98 errors:0 dropped:0 overruns:0 carrier:0
     collisions:0 txqueuelen:0 
     RX bytes:15640 (15.2 KiB) TX bytes:15873 (15.5 KiB)

さらにOpen vSwitchでもブリッジの設定が行われているか確認します。
    
    
    # ovs-vsctl show

これでbr-exの項目にPortとしてeth0が表示されていれば問題ありません。

次にOpenStack側でpublicネットワーク、adminテナントでのprivateネットワークの作成を行います。コマンドからも可能ですが、今回はHorizonから行います。

まず「管理→システムパネル→ネットワーク」から「ネットワークの作成」ボタンを押してネットワークを追加するためのダイアログを表示し、下記のような内容を入力後に「ネットワークの作成」ボタンを押します。

  * 名前：public(任意の名称で問題ありません)
  * プロジェクト：admin
  * 管理状態：チェック
  * 共有：チェック
  * 外部ネットワーク：チェック



これで新たにpublicという名前のネットワークが追加されます。

次にpublicのリンクをクリックし、そのネットワークのサブネットを作成するために「サブネットの作成」ボタンをクリックします。サブネットを作成するためのダイアログが表示されたら下記のような内容を入力します。

  * サブネット名：public_subnet(任意の名称で問題ありません)
  * ネットワークアドレス：br-exで利用しているネットワークアドレス。今回の場合は192.168.56.0/24
  * ゲートウェイIP：今回の場合192.168.56.1
  * DHCP有効：チェック
  * IPアドレス割り当てプール：既存のDHCPサーバーで割り当てる範囲とかぶらない範囲で割り当てます



最後に作成ボタンを押すとpublic_subnetが作成されます。

次にプロジェクトで利用するprivateネットワークを作成します。「プロジェクト→ネットワーク→ネットワーク」から「ネットワーク作成」ボタンを押すとネットワーク作成のためのダイアログが表示されるので下記内容を最後まで入力し「作成」ボタンを押します。先ほどと異なりサブネットも同時に作るようなダイアログになっています。

  * 名前：admin_private(任意の名称で問題ありません)
  * 管理状態：チェック
  * サブネット名：admin_private_subnet
  * ネットワークアドレス：任意のプライベートIPアドレスの範囲。今回は192.168.111.0/24
  * ゲートウェイIP：192.168.111.1(後ほど作成するルーターのIPアドレスになります)
  * DHCP有効：チェック



これでadminテナントで利用できるプライベートネットワークが作成されました。最後に「プロジェクト→ネットワーク→ルーター」からadmin_privateからpublicへネットワークをつなげるルーターを作成します。「ルーターの作成」ボタンを押すとルーターの名前を聞かれるので今回は「admin_router」とし、「ルーターの作成」ボタンを押すとルーターが追加されます。

作成されたルーターの一番右の「ゲートウェイの設定」ボタンを押し、ルーターのゲートウェイを設定します。出てきたダイアログの外部ネットワークで「public」を選び「ゲートウェイの設定」ボタンを押します。するとルーター一覧の外部ネットワークの部分がpublicになります。

最後にルーターをadmin_privateに接続します。ルーターの名称のリンクをクリックし、ルーターの詳細画面を表示します。その中で「インターフェースの追加」ボタンを押し、サブネットからadmin_private_subnetを選択し「インターフェースの追加」ボタンを押して完了です。

確認のために「プロジェクト→ネットワーク→ネットワークトポロジ」から確認するとadmin_privateからpublicへadmin_routerを介して接続されている様子が確認できます。これでネットワークの設定は完了です。

## インスタンスの起動とFloating IPの割当による外部からのアクセス

インスタンスが正しく起動するかチェックを行う際には軽量なCirrOSが使われますが、今回はCentOSを利用します。こちらもコマンドからも可能ですが、今回はHorizonから行います。

ログイン後に「管理→システムパネル→イメージ」からイメージの管理画面を出します。画面内の「イメージ作成」ボタンを押すとダウンロードするイメージの入力ダイアログが表示されるので下記のように入力し「イメージの作成」を押します。

  * 名前：CentOS6(なんでも良いです)
  * イメージソース：イメージの場所
  * イメージの場所：http://repos.fedorapeople.org/repos/openstack/guest-images/centos-6.5-20140117.0.x86_64.qcow2
  * 形式：qcow2
  * パブリック：チェックをつける



これでしばらく待つとイメージの状態がActiveとなり利用できるようになります。これでOpenStackでCentOSが利用できるようになりました。

[![add_centos_image]({{ site.baseurl }}/assets/スクリーンショット-2014-05-14-1.26.261-300x79.png)](http://blog.hirokikana.com/wp-content/uploads/2014/05/スクリーンショット-2014-05-14-1.26.261.png)次にCentOSのインスタンスを立ち上げる準備を行います。立ち上げたインスタンスはsshでログインする際にパスワードでログインすることはできず、鍵を使った認証でログインをします。そのために自分のマシンの公開鍵をOpenStackに登録します。「プロジェクト→コンピュート→アクセスとセキュリティ」の画面にある「キーペア」をクリックし「キーペアのインポート」から自分の公開鍵を設定することができます。自分のマシンの鍵を利用したく無かったり、このシステムで専用の鍵を利用したい場合は隣の「キーペアの作成」ボタンを押すと新しい鍵を作成し秘密鍵がブラウザ経由でダウンロードできます。用途に合わせて好きな方法で鍵を設定してください。

[![add_key_pair]({{ site.baseurl }}/assets/スクリーンショット-2014-05-14-1.34.35-300x71.png)](http://blog.hirokikana.com/wp-content/uploads/2014/05/スクリーンショット-2014-05-14-1.34.35.png)

次に同じ画面の「セキュリティグループ」をクリックし、仮想マシンに対してアクセスできる通信に関する設定を行います。これを設定しない場合pingすら仮想マシンへ対して到達することができません。今回は元からあるdefaultのセキュリティグループを編集するので名前defaultのルールにある「ルールの管理」ボタンをクリックします。その画面で「ルールの追加」ボタンを押しルールの中から「SSH」を選択し「追加」ボタンを押します。これでSSHで疎通ができるようになりました。同様の方法でpingが通るようにALL ICMP、HTTPサーバーを立ち上げるならHTTPもルールとして足しておきましょう。

[![add_security_group_rule_ssh]({{ site.baseurl }}/assets/スクリーンショット-2014-05-14-1.39.05-300x247.png)](http://blog.hirokikana.com/wp-content/uploads/2014/05/スクリーンショット-2014-05-14-1.39.05.png)さて、いよいよインスタンスの立ち上げを行います。「プロジェクト→コンピュート→インスタンス」の画面から「インスタンスの起動」ボタンで起動するインスタンスの設定を行います。

  * 詳細→インスタンス名：任意の名前。ホスト名にもなります。
  * 詳細→フレーバー：何故かCentOSだとm1.tinyだと立ち上がらなかったのでm1.smallを選択
  * 詳細→インスタンスのブートソース：イメージから起動
  * 詳細→イメージ名：先ほど保存したCentOSのイメージ
  * アクセスとセキュリティ→キーペア：先ほど作成したキーペアの名前
  * アクセスとセキュリティ→セキュリティグループ：defaultにチェック
  * ネットワーク：admin_privateの右下の「+」ボタンを押して選択済みネットワークにする



[![instance_setting]({{ site.baseurl }}/assets/スクリーンショット-2014-05-14-1.45.43-300x276.png)](http://blog.hirokikana.com/wp-content/uploads/2014/05/スクリーンショット-2014-05-14-1.45.43.png)

設定後に「起動」ボタンを押すとインスタンスの作成が開始されます。稼動状態がRunningになったら正常に起動しています。正常に起動しているのを確認したら、インスタンス名をクリックしインスタンスの詳細を表示します。インスタンスの詳細ではコンソールを表示したり、ログ(dmesgの内容)を閲覧することができます。

[![instance_console]({{ site.baseurl }}/assets/スクリーンショット-2014-05-14-1.49.15-300x172.png)](http://blog.hirokikana.com/wp-content/uploads/2014/05/スクリーンショット-2014-05-14-1.49.15.png)

次にFloating IPを割り当ててホストOSからアクセスできるようにします。「プロジェクト→コンピュート→アクセスとセキュリティ」の「Floationg IP」から「Floating IPの確保」ボタンを押し、Floating IPの確保を行うダイアログを表示します。プールからpublicを選択し「IPの確保」を押すとFloting IPが確保されます。

[![add_floating_IP]({{ site.baseurl }}/assets/スクリーンショット-2014-05-14-1.53.26-300x120.png)](http://blog.hirokikana.com/wp-content/uploads/2014/05/スクリーンショット-2014-05-14-1.53.26.png)

割り当てたいFloating IPの右側にある「割り当て」ボタンを押すとそのFloating IPの割り当てダイアログが表示されるので、IPを割り当てるポートで先ほど起動したインスタンスを選択し「割り当て」ボタンを押します。これでFloating IPを割り当てることができました。

[![setting_floating_IP]({{ site.baseurl }}/assets/スクリーンショット-2014-05-14-1.55.27-300x139.png)](http://blog.hirokikana.com/wp-content/uploads/2014/05/スクリーンショット-2014-05-14-1.55.27.png)

割り当てられたかどうかは「プロジェクト→コンピュート→インスタンス」で表示される表のインスタンスのIPアドレスにFloating IPで割り当てたIPアドレスが追加されていることを確認します。また、この状態で割り当てたIPアドレスに対してホストOSでpingをし疎通することを確認します。

ホストマシンから下記のようにアクセスし、ログインすることができればあとは通常のCentOSと同じことが可能です。CentOSのイメージではユーザーがcloud-userで利用するようになっています。
    
    
    $ ssh -i <設定したsshキーファイル> cloud-user@<割り当てたIPアドレス>

## まとめ

今回はCentOS6.5を利用し、OpenStackの最新版であるIcehouseをインストールしました。実はこの投稿を書き始めていた段階では https://bugs.launchpad.net/nova/+bug/1298640 この問題を踏んだりしていて、そのことも書こうと画策していました。しかしながら、書きながら何度か動作確認する中でpackstackの方で対策が行われており、活発な開発が行われていることを実感しました。またpackstackはRedHat系のOSで投入するときはぜひおすすめです。

ここまででようやくOpenStackのインストールにも慣れてきたので同一の手法で物理マシンに構築し早速利用しています。OpenStackには簡単に仮想マシンを用意できるだけでなく便利な機能がたくさんあるので随時投稿していきたいと思います。
