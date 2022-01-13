---
layout: post
title: Kolla Ansible を利用して VirtualBox 上の仮想マシンに OpenStack をデプロイする
date: 2022/01/13 22:50:00
type: post
published: true
status: publish
categories:
 - dev
tags: 
 - OpenStack
 - Kolla Ansible
---
## Kolla Ansible とは
Kolla Ansible とは OpenStack の各サービスやコンポーネントを Docker コンテナの形で導入する仕組みです。

もともとは [OpenStackコンポーネントをKubernetes上に構築　Yahoo! JAPAN巨大インフラの運用と舞台裏 - Part1 - ログミーTech](https://logmi.jp/tech/articles/320926) この OpenStack のコンポーネントをコンテナにし Kubernetes で管理している仕組みをみて、自前でコンテナイメージを作ろうとしていました。

その中で、 Kolla Ansible の存在を知り「これでいいじゃん」となり利用してみたという経緯です。
実際に利用してみたところ(手動導入時と比べて)非常に簡単に OpenStack を導入することができたため、利用方法の備忘録として残すことにしました。

## 環境
私は色々考える中で物理マシンを買いがちなのですが、色々試すという意味ではやはり仮想マシンで作成するのが本当に楽です。
NIC やディスクが手軽に追加できたりしますしね。

今回は macOS 上の VirtualBox を vagrant から利用して検証をしました。
理屈上、 Windows でも動くとは思うんですが、検証してないのとちょっとした癖があってこのままの手順ではないかもしれません。

## VirtualBox 特有の設定等
VirtualBox 特有の設定としては下記があります。
 - Nested Virtualization を有効にする
 - それぞれの NIC でプロミスキャスモードを許可する
 - ディスクのサイズを拡張する

最近の VirtualBox のバージョンにおいては、Intel / AMD どちらのプロセッサにおいても Nested Virtualization が利用できます。
そのため、各仮想マシンにおいては Nested Virtualization を有効にしておきましょう。
```Vagrantfile
  config.vm.provider "virtualbox" do |vb|
(略)
    vb.customize ["modifyvm", :id, "--nested-hw-virt", "on"]
```

プロミスキャスモードですが、許可されていない場合 Floating IP アドレスが利用できない状態になります。
そのため、仮想マシンの設定において各 NIC のプロミスキャスモードを許可しておきます。
```Vagrantfile
    vb.customize ["modifyvm", :id, "--nicpromisc2", "allow-all"]
    vb.customize ["modifyvm", :id, "--nicpromisc3", "allow-all"]
```

ディスクのサイズは何もしないと 10 GB で作成されます。
公式ドキュメント (参考リンク[1]) の Host machine requirements にあるように 40GB に変更しておきます。
```Vagrantfile
    server.vm.disk :disk, size: "40GB", primary: true
```
上記をしていした場合、 `vagrant up` をする際に環境変数において明示的に有効化する必要があります。
起動の際は `VAGRANT_EXPERIMENTAL="disks" vagrant up` と指定することで有効になります。(参考リンク[2])
また、起動したあとにおいて OS 上で拡張する必要もあるため、Provisioning からシェルを実行し拡張するようにします。
```Vagrantfile
  config.vm.provision "shell", inline: <<-SHELL
    yum update -y ; yum install -y openssh-server python3-devel libffi-devel gcc openssl-devel python3-libselinux git python3-pip cloud-utils-growpart
(略)
    growpart /dev/sda 1
    xfs_growfs -d /
```

上記の内容を含めた Vagrantfile は下記です。
```Vagrantfile
# -*- mode: ruby -*-
# vi: set ft=ruby :
Vagrant.configure("2") do |config|
  config.vm.box = "centos/stream8"

  config.vm.define "ctrl01" do |server|
    server.vm.hostname = "ctrl01"
    server.vm.network "private_network", ip: "192.168.56.100"
    server.vm.network "private_network", type: "dhcp", auto_config: false, name: 'vboxnet1'
    server.vm.disk :disk, size: "40GB", primary: true
  end
  config.vm.define "comp01" do |server|
    server.vm.hostname = "comp01"
    server.vm.network "private_network", ip: "192.168.56.101"
    server.vm.network "private_network", type: "dhcp", auto_config: false, name: 'vboxnet1'
    server.vm.disk :disk, size: "40GB", primary: true
  end
  config.vm.define "comp02" do |server|
    server.vm.hostname = "comp02"
    server.vm.network "private_network", ip: "192.168.56.102"
    server.vm.network "private_network", type: "dhcp", auto_config: false, name: 'vboxnet1'
    server.vm.disk :disk, size: "40GB", primary: true
  end

  config.vm.provider "virtualbox" do |vb|
    vb.memory = "8192"
    vb.customize ["modifyvm", :id, "--nested-hw-virt", "on"]
    vb.customize ["modifyvm", :id, "--nicpromisc2", "allow-all"]
    vb.customize ["modifyvm", :id, "--nicpromisc3", "allow-all"]
  end
  
  config.vm.provision "shell", inline: <<-SHELL
    yum update -y ; yum install -y openssh-server python3-devel libffi-devel gcc openssl-devel python3-libselinux git python3-pip cloud-utils-growpart
    echo "PermitRootLogin yes" >> /etc/ssh/sshd_config
    sed -i s/PasswordAuthentication\\ no/PasswordAuthentication\\ yes/ /etc/ssh/sshd_config
    systemctl restart sshd
    growpart /dev/sda 1
    xfs_growfs -d /
    dnf remove -y python3-pyyaml
    pip3 install -U pip
    pip3 install 'ansible<5.0'
    ip addr flush eth2
  SHELL
end
```
その他にも sshd でパスワード認証を有効にしたり、Ansible を導入したりして定型的に処理できる部分はここで実施されるようにしておきます。
上記の構成ではコントローラーノード 1 台とコンピュートノード 2 台が作成されます。
手元の環境 (Mac mini 2018 / Mem 32 GB) ではこれが全部立ち上がるとメモリリソースが結構厳しい感じになりました。

## コンピュートノードにパスワードなしでログインできるようにする
コンピュートノードに対しては Ansible で導入を実行するわけですが、その際パスワード認証を利用していると毎回パスワードを聞かれてしまい、作業になりません。
そのため、コンピュートノードには鍵認証でログインできるようにしておきます。

まず、root で SSH ログインをできるようにしておきます。
デフォルトでは root ログインもパスワード認証も許可されていないので、sshd の設定を変更する必要がありますが、そこは Provisioning の際の shell で実行しておきます。
```Vagrantfile
    echo "PermitRootLogin yes" >> /etc/ssh/sshd_config
    sed -i s/PasswordAuthentication\\ no/PasswordAuthentication\\ yes/ /etc/ssh/sshd_config
    systemctl restart sshd
```

このうえで、各コンピュートノードで root のパスワードも設定しておきます。
```
vagrant ssh comp01
sudo passwd
```

そのうえで、コントローラーノードの root の公開鍵を登録し、パスワードなしでログインできるようにします。
```
vagrant ssh ctrl01
sudo su - 
ssh-keygen
ssh-copy-id root@192.168.56.101 #comp01
ssh-copy-id root@192.168.56.102 #comp02
```

最後にそれぞれのコンピュートノードにパスワードなしでログインすることが可能であることを確認しておきます。

## Kolla Ansible を利用してセットアップ
基本的には Quick Start (参考リンク[1]) の内容のとおりではあるのですが、手元で検証した際の流れを紹介します。
Python の venv は利用せず、 OS に予め導入されている Python を利用しました。
そのため、 Quick Start を参照される際は `If not using a virtual environment:` とある方の手順を進めていく感じです。

Ansible の導入までは Provisioning で終了しているので Quick Start の `Install Kolla-ansible` の内容を進めていきます。
手順をすすめるにあたり、私は `sudo su -` して root になって作業をしました。
```
pip3 install git+https://opendev.org/openstack/kolla-ansible@master
mkdir -p /etc/kolla
chown $USER:$USER /etc/kolla
cp -r /usr/local/share/kolla-ansible/etc_examples/kolla/* /etc/kolla
cp /usr/local/share/kolla-ansible/ansible/inventory/* .
```
上記を実行すると、Kolla Ansible が導入され、必要な設定ファイルが `/etc/kolla` 以下に、カレントディレクトリにノードの設定を行う `all-in-one` と `multinode` のファイルがコピーされます。

まず、ノードの設定では `multinode` を利用します。
`multinode` においては `[compute]` で今回作成した `192.168.56.101` / `192.168.56.102` を指定し、その他の項目については `localhost` を指定します。
```multinode
[control]
localhost       ansible_connection=local

[network]
localhost       ansible_connection=local

[compute]
192.168.56.101
192.168.56.102

[monitoring]
localhost       ansible_connection=local

[storage]
localhost       ansible_connection=local

[deployment]
localhost       ansible_connection=local
(略)
```

次に Kolla Ansible の設定を行うため、 `/etc/kolla/globals.yml` を編集します。
ここでは変更点だけ紹介します。

```
kolla_base_distro: "centos"
kolla_install_type: "binary"
```
まず `kolla_base_distro` は CentOS であるため、`centos` を指定します。
`kolla_install_type` ですが、デフォルトでは `source` でドキュメントによると `source` のほうが信頼性が高いとのことです。
私は、きっと `binary` のほうがビルドとかたぶんしないんじゃないかという根拠のない予想から、早く作業が完了しそうと考え `binary` にしました。(実際のところは測定していません)

```
kolla_internal_vip_address: "192.168.56.100"
network_interface: "eth1"
neutron_external_interface: "eth2"
```
`kolla_internal_vip_address` は `network_interface` で指定した NIC の IP アドレスを指定します。
`network_interface` は、今回の構成でいうところの Management network で利用する NIC を指定します。
作成された VM の eth0 はインターネットと疎通させるために NAT の NIC にしているので、今回の場合は eth1 を指定します。
`neutron_external_interface` は Neutron でパブリックネットワークと接続する NIC を指定するわけですが、今回の場合は eth2 を指定します。
ネットワーク構成の名称と役割については Oracle Openstack の Configraion Guide にわかりやすい図がありました。(参考リンク [3])

```
enable_haproxy: "no"
```
これはちょっと深堀りできていないのですが、 `enable_haproxy` が `yes` の場合失敗したので `no` にしました。
おそらく複数のコントローラーノードのときに利用するものであると認識しているのですが、よく調べていないので間違っているかもしれません。
ただ、今回利用する範囲では `no` にしても支障はなかったです。

Quick Start には `enable_cinder` に関して記載がありますが、今回は Cinder を利用せず最小限の構成で動作をさせます。

ここまで終わればあとは下記のコマンドを実行し、すべての Ansible の処理が正常に完了すれば導入完了です。
```
kolla-genpwd
kolla-ansible -i ./multinode bootstrap-servers
kolla-ansible -i ./multinode prechecks
kolla-ansible -i ./multinode deploy
```
`deploy` が一番時間がかかるのでしばらく放置しておきます。

## 実際に OpenStack を利用する
最後に admin ユーザの作成等を実施します。
まず作業を始める前に OpenStack Client が必要となるため、インストールします。
```
pip install python-openstackclient -c https://releases.openstack.org/constraints/upper/master
```
次に admin ユーザの作成等の作成を実施します。
```
kolla-ansible post-deploy
```

この状態で `. /etc/kolla/admin-openrc.sh` すると OpenStack が利用できる形となります。
また、`http://192.168.56.100` にアクセスすると Horizon のログイン画面が表示されると思います。
ユーザ名に `admin` パスワードには `/etc/kolla/admin-openrc.sh` 内の `OS_PASSWORD` に設定されている値を利用し、ログインが可能になっていると思います。

最後に動作確認のための OS イメージやネットワークを `/usr/local/share/kolla-ansible/init-runonce` を利用して用意します。
ただ、このファイルを実行すると外部ネットワークの IP アドレスが異なるので、手元にコピーし編集したうえで実行します。
```init-runonece
EXT_NET_CIDR=${EXT_NET_CIDR:-'192.168.56.0/24'}
EXT_NET_RANGE=${EXT_NET_RANGE:-'start=192.168.56.170,end=192.168.56.199'}
EXT_NET_GATEWAY=${EXT_NET_GATEWAY:-'192.168.56.1'}
```

最後に `openstack server create --image cirros --flavor m1.tiny --key-name mykey --network demo-net demo1` を実行するとインスタンスが作成されます。
作成されたインスタンスは Floating IP アドレスを付与することで VM のホストマシンからも疎通できることが確認できると思います。


## まとめ
OpenStack の導入はコマンドひとつで導入できず、時間や手間、導入の段階でトラブルシューティングが必要であったかと思います。
しかし、この方法であれば割と簡単に導入することができました。
また、それぞれのモジュールが Docker コンテナの形で動作するため、必要なくなったらコンテナを削除するだけできれいな環境に戻すことができたりする点も利点かと思います。

## 参考リンク
 - [1] [Quick Start kolla-ansible 13.1.0.dev131 documentation](https://docs.openstack.org/kolla-ansible/latest/user/quickstart.html)
 - [2] [Vagrant Disk Usage  Vagrant by HashiCorp](https://www.vagrantup.com/docs/disks/usage)
 - [3] [Configuring Network Interfaces for OpenStack Networks](https://docs.oracle.com/cd/E90981_01/E90982/html/kolla-openstack-network.html)
