---
layout: post
title: VirtualBox上にOpenStack(Grizzly)をインストールする
date: 2013-11-09 22:16:02.000000000 +09:00
type: post
published: true
status: publish
categories:
- dev
tags:
- OpenStack
meta:
  _edit_last: '1'
  _thumbnail_id: '439'
  _wp_old_slug: virtualbox%e4%b8%8a%e3%81%abopenstackgrizzly%e3%82%92%e3%82%a4%e3%83%b3%e3%82%b9%e3%83%88%e3%83%bc%e3%83%ab%e3%81%99%e3%82%8b
author:
  login: admin
  email: hiroki@hirokikana.com
  display_name: hiroki.kana
  first_name: Hiroki
  last_name: Takayasu
---
OpenStackはオープンソースのクラウド・インフラ構築のソフトウェアです。最近ではクラウド・インフラとしてAmazon EC2などがすっかり一般的となりましたが、開発として使うには従量課金であることもあり、手軽に使えるとは言えない状況にあると思います。そのようなときにOpenStackのような自前のハードウェアにインストールすることで自分専用のクラウド・インフラを用意することができます。「極力お金はかけたくないけど試しにEC2っぽいものを触ってみたい」という人にOpenStackは最適だと思います。

とはいえ、わざわざ検証用の物理マシンを用意することができるかというと、そんなことも無いと思います。そこでVirtualBox上に構築することで手軽にOpenStackを試すことができます。新たに物理マシンを用意する必要もありませんし、ノートPC内にインストールすれば手軽に環境を持ち運ぶこともできます。なにより仮想マシンはスナップショットがあるので、うまくいかなかったらうまくいっていた場所まで状態を戻すこともできます。

今回はVirtualBox上に検証用のOpenStackをインストールする手順を紹介します。

### はじめに

まず手順については下記サイトをかなり参考にさせていただきました。というか、手順のほとんどは下記のサイトと同一です。ありがとうございます。

  * [最短手順でRDO(Grizzly)のデモ環境を構築 - めもめも](http://d.hatena.ne.jp/enakai00/20131022/1382443408 "最短手順でRDO\(Grizzly\)のデモ環境を構築 - めもめも" )



検証は下記のような環境で行いました。

  * MacBook Pro Retina,Mid 2012
  * Intel Core i7 2.3GHz
  * 8GB 1600MHz DDR3 Memory
  * Mac OS X 10.8.5 Mountain Lion



### VirtualBoxで仮想マシンの作成

まずはVirtualBoxで仮想マシンを作成します。一般的な項目についてはひとまず飛ばして特筆すべきところだけを書きます。

まずメモリはある程度容量大きめに確保しておきたいところです。とはいえ、実際に常用するわけではないので2〜4GB程度を割り当てておけば検証には十分でしょう。

ストレージですが割と大きめに作成します。可変サイズストレージであれば大きめに確保しても即容量が消費されるわけではないので、最初のうちに大きめにしておきましょう。ちなみに今回は120GBのものを作成しました。

[![OpenStackVIrtualBox_vmdisk_setting]({{ site.baseurl }}/assets/スクリーンショット-2013-11-09-2.37.35-300x242.png)](http://blog.hirokikana.com/wp-content/uploads/2013/11/スクリーンショット-2013-11-09-2.37.35.png)

作成ウィザード完了後にネットワークの設定を行います。今回は参考サイトの例の通り2つのNICを割り当てます。アダプタの種類ですが、今回はブリッジアダプタを利用しました。ここで設定しておく必要があるのがNICの**「プロミスキャスモード」を「すべて許可」**にしておく必要があります。これを行わないと構築後に仮想マシンとFloatingIP経由で通信することができませんでした。(この設定が漏れてて、1週間以上ハマってました…)

[![OpenStackVIrtualBox_vmnic_setting]({{ site.baseurl }}/assets/スクリーンショット-2013-11-09-2.39.44-300x242.png)](http://blog.hirokikana.com/wp-content/uploads/2013/11/スクリーンショット-2013-11-09-2.39.44.png)

以上で仮想マシンの設定が完了です。

### Fedora18のインストールとOpenStackのインストール

次にFedora18をインストールしOpenStackをインストールします。OpenStackのインストール方法にはいくつかあるんですが、今回は最も手軽かつRedHat系にインストールできるということでPackstackを利用した方法でインストールします(OpenStackにインストールする際の多くはUbuntuが利用されていますが、私はRedHat系のOSを使いたかったので…)。RedHat系のOSにOpenStackを構築するためのコミュニティとして[RDO](http://openstack.redhat.com/Main_Page "RDO" )というものがあり、ドキュメントなどもかなり充実しているのでぜひ一度目を通しておくことをおすすめします。

さてそれではまずはFedora18をインストールします。ここでも特筆すべきところだけを記載します。

まずネットワークの設定ですが、見慣れないpXpXのようなインターフェイス名になっています。これはFedora15からethXのようなインターフェイス名からどこにあるデバイスかしっかりわかる形にするための変更が入った影響です。例えばp2p1はPCIスロット番号2のポート番号1番のデバイスであることを示しています。今回作成時にはp2p1とp7p1ができたので、p2p1にはマシンに割り当てるIPアドレス設定を行います。p7p1に関してはインストール後に設定ファイルからIPアドレスを付与しない形で活性化しておきます。/etc/sysconfig/network-scripts/ifcfg-p7p1の下記項目をインストール後に編集します。
    
    
    ONBOOT=no

ソフトウェアの選択は「最小限のインストール」を選択しアドオンの「標準」にチェックを入れます。余分なものはインストールしないようにします。

[![Openstack_vm_fedora_instal_select_package]({{ site.baseurl }}/assets/スクリーンショット-2013-11-01-22.32.17-300x239.png)](http://blog.hirokikana.com/wp-content/uploads/2013/11/スクリーンショット-2013-11-01-22.32.17.png)

ディスクはCinderというVMで利用するイメージを管理するソフトウェアでLVMで割当られた領域を利用します。そのためcinder-volumesという名称のボリュームグループを作成します。最終的には下記のようなレイアウトにします。

[![openstack_vm_fedora_partition_setting]({{ site.baseurl }}/assets/スクリーンショット-2013-11-03-21.04.16-300x243.png)](http://blog.hirokikana.com/wp-content/uploads/2013/11/スクリーンショット-2013-11-03-21.04.16.png)

 

最後にインストール中にrootパスワードの設定も忘れずにやっておきましょう。

[![openstack_demo_root_passwd_setting]({{ site.baseurl }}/assets/スクリーンショット-2013-11-01-22.33.55-300x239.png)](http://blog.hirokikana.com/wp-content/uploads/2013/11/スクリーンショット-2013-11-01-22.33.55.png)

インストールが完了し再起動したら、インストールスクリプトをダウンロードしsetup.shを実行します。setup.shでは必要になるものを自動的にインストールしPackStackを利用しall-in-one構成でOpenStackをインストールしてくれます。
    
    
    # yum -y install git
    # cd ~
    # git clone https://github.com/enakai00/quickrdo
    # cd quickrdo
    # git checkout f18-grizzly
    # ./setup.sh

途中でrootパスワードを聞かれるので入力します。その後約20分程度でセットアップが完了します。完了後にOpenStackでどのハイパーバイザーを利用するか設定するために、/etc/nova/nova.confの下記のように編集します。
    
    
    libvit_type=qemu

デフォルトではkvmになっていると思いますが、仮想マシン上なのでKVMは動作しないのでqemuに変更しQEMUによるエミュレーションを行うようにします。QEMUによる仮想マシンはパフォーマンスの面でかなり劣るのであくまで検証用に利用するという目的になります。

ここで一度マシンを再起動します。

次にquickrdoに含まれているconfig.shでOpenStackのアカウントやネットワークの設定を行います。設定を行うにあたり、このタイミングでは下記を決めておく必要があります。

  * public：インターネットへ繋がるネットワークのネットワークアドレス。
  * gateway：デフォルトゲートウェイ
  * nameserver：DNSサーバー
  * pool：publicで指定したネットワークでOpenStack内で利用するIPアドレスの範囲。とりあえずチェックするだけであれば20個程度確保できる範囲を指定すれば十分です
  * private：仮想マシンのみが通信するためのネットワーク



それぞれについてquickrdo/config.sh内に設定します。設定を書いたら次のコマンドを実行し設定を行います。
    
    
    # cd ~/quickrdo
    # ./config.sh

実行直後に「[VM](http://d.hatena.ne.jp/keyword/VM) [access](http://d.hatena.ne.jp/keyword/access) [NIC](http://d.hatena.ne.jp/keyword/NIC):」と聞かれるのでIPアドレスの設定されていないp7p1を指定します。

これでFedora18とOpenStackのインストールは完了です。

### OpenStackの利用

初期設定スクリプトでは下記のユーザーが自動的に作成されているので、一般ユーザーでログインします。URLはインストール時に設定したIPアドレスです。

ユーザ名 | パスワード | ロール  
---|---|---  
demo_user | passw0rd | 一般ユーザー  
demo_admin | passw0rd | 管理者  
  
ログイン後に先ほどの初期設定スクリプトで登録されたFedora19のイメージを使って新しいインスタンスを作成します。まず左側のメニューの「インスタンス」を選択しインスタンス画面を開いたあとに「インスタンスの起動」ボタンを押します。

[![openstack_instance]({{ site.baseurl }}/assets/スクリーンショット-2013-11-09-19.12.16-300x176.png)](http://blog.hirokikana.com/wp-content/uploads/2013/11/スクリーンショット-2013-11-09-19.12.16.png)インスタンスの設定を行う画面がウィザードが表示されるのでイメージで「Fedora19」を選択し、適当なインスタンス名を入力します。

[![openstack_instance_select_image]({{ site.baseurl }}/assets/スクリーンショット-2013-11-09-19.14.38-300x275.png)](http://blog.hirokikana.com/wp-content/uploads/2013/11/スクリーンショット-2013-11-09-19.14.38.png)次にネットワークタブを選択し、利用可能なネットワークからprivate01を青い選択済みネットワークの部分にドラッグアンドドロップし、選択します。

[![openstack_network_setting]({{ site.baseurl }}/assets/スクリーンショット-2013-11-09-19.16.06-300x166.png)](http://blog.hirokikana.com/wp-content/uploads/2013/11/スクリーンショット-2013-11-09-19.16.06.png)他にも設定は可能なのですが、ひとまずこれで「起動」ボタンを押して起動します。すると、インスタンス画面に作成したインスタンスが1つ増えます。最初は状態がBuildになっていますが、しばらくするとActiveになりインスタンスが利用可能になります。

[![openstack_instance_boot]({{ site.baseurl }}/assets/スクリーンショット-2013-11-09-19.41.00-300x234.png)](http://blog.hirokikana.com/wp-content/uploads/2013/11/スクリーンショット-2013-11-09-19.41.00.png)インスタンスが起動したらFloating IPを割り当てます。Floationg IPはインスタンス同士しかやりとりできないプライベートネットワークに対して外部とやり取りできるネットワークのIPアドレスを関連づけし、外部のネットワークを利用できるようにする仕組みです。インスタンス右側の「さらに」から「Floating IPの割り当て」を選び、IPアドレスから適当なIPアドレスを選択します。

[![openstack_floating_ip]({{ site.baseurl }}/assets/スクリーンショット-2013-11-09-21.46.16-300x146.png)](http://blog.hirokikana.com/wp-content/uploads/2013/11/スクリーンショット-2013-11-09-21.46.16.png)これで割り当てたFloating IPを利用することができます。

次にインスタンスにSSHでログインします。ログインするには/rootにあるmykey.pemを利用しログインします。
    
    
    #  ssh -i mykey.pem fedora@<Floating IPで割り当てたIPアドレス>

これでOpenStackの構築は完了です。

### まとめ

仮想マシンに構築したOpenStack内のインスタンスでなんらかの開発を行うのはパフォーマンスの面で非常に不利になると思います。しかし、仮想マシン内に手軽にデモ、検証環境を構築できることでどこでもOpenStack自体の動作を確認する目的としては非常に有用だと思います。いきなり物理マシンに構築するのではなく、まず仮想マシンでテストし正しく動作することを確認してから物理マシンに構築できるようになるというのは安心感もあります。

今回構築していくなかで何よりはまったのはネットワークアダプタのプロミスキャスモードの設定でした。最初VirtualBoxに構築しネットワークが疎通しなかったので、スクリプトではなく手動で構築したり、余った物理マシンで構築して原因を探るのにかなり時間がかかりました…。そもそもどのようにネットワークが疎通しているかを考えればプロミスキャスモードに考えが至るのは自然だと思うので、そこまで至らなかった自分はまだまだ勉強不足というかなんというか…。同じことではまってる方がいて、この記事を見て解決していただけたらうれしいです。
