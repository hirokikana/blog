---
layout: post
title: OpenStack(Icehouse)にpackstackでcomputeノードを追加
date: 2014-05-27 02:32:29.000000000 +09:00
type: post
published: true
status: publish
categories:
- dev
tags:
- OpenStack
meta:
  _edit_last: '1'
  _thumbnail_id: '502'
  _wp_old_slug: openstackicehouse%e3%81%abpackstack%e3%81%a7compute%e3%83%8e%e3%83%bc%e3%83%89%e3%82%92%e8%bf%bd%e5%8a%a0
author:
  login: admin
  email: hiroki@hirokikana.com
  display_name: hiroki.kana
  first_name: Hiroki
  last_name: Takayasu
---
[前回](http://blog.hirokikana.com/?p=461)はCentOS 6.5上にOpenStack(Icehouse)をオールインワン構成で構築しました。OpenStackの各モジュールは複数台にスケールすることが予め考慮されており、実際に利用する際にはそれぞれのモジュールを別サーバーに構築するということになることもあるかと思います。今日は数多くのモジュールの中で実際に仮想マシンを起動するcomputeノードをpackstackを使って追加してみたいと思います。

## 構築する環境のおさらい

構築するのは前回の記事で作成したVirtualBox上のオールインワン構成のOpenStackにcomputeノードを足します。OpenStackの各マシン同士が疎通するためのネットワークとFloating IPを割り当てて疎通させるNICを分けたりしている構成をよく見るのですが、今回はひとまず同じNICを利用するようにします。OSはpackstackを使う都合から同じCentOS 6.5にします。

[caption id="attachment_502" align="aligncenter" width="728"][![OpenStack_add_comupte_node]({{ site.baseurl }}/assets/OpenStack_add_comupte_node.png)](http://blog.hirokikana.com/wp-content/uploads/2014/05/OpenStack_add_comupte_node.png) 追加するComputeノードの構成[/caption]

## computeノードに利用するマシンの用意

今回は前回構築したオールインワン環境に新たにcomputeノードを1台追加します。仮想マシンは下記のような内容で作成をします。

  * ネットワークアダプタ1：ホストオンリーアダプタ、プロミスキャスモードをすべて許可
  * ネットワークアダプタ2：NAT
  * OS：CentOS 6.5最小構成
  * SELinux：permissive



メモリやCPUのコア数はホストOSのリソース状況を見て適当に設定をしてください。私はCPUコア数4、メモリ2048MBで構成しました。このマシンもVirtualBox上に構築するためハイパーバイザーはqemuを利用します。インストールに際しては特に特別なものを投入することなく最小構成でインストールします。

インストール後にネットワークアダプタ1のIPアドレスは前回構築したOpenStackと疎通できるようにしておきます。前回eth0に192.168.56.20を割り当てましたが、新しく用意したcomputeノード用のマシンには192.168.56.21を割り当てました。割り当て後前回構築したOpenStackから疎通することを確認しておきましょう。
    
    
    # ping 192.168.56.20
    PING 192.168.56.20 (192.168.56.20) 56(84) bytes of data.
    64 bytes from 192.168.56.20: icmp_seq=1 ttl=64 time=0.044 ms
    64 bytes from 192.168.56.20: icmp_seq=2 ttl=64 time=0.032 ms

またNAT経由でインターネットと疎通するかも同時に確認しておきましょう。疎通しない場合はeth1が有効になっているか再度確認してください(ifcfg-eth1のONBOOTがデフォルトnoになっているのでインストール時もしくは起動後に設定しないと有効になりません)。

## controllerノードからpackstackでcomputeノードの追加

まず、前回構築したOpenStack(controllerノードと呼びます)のanswerファイルを開き、CONFIG_NOVA_COMPUTE_HOSTSに下記のようにカンマ区切りで新たに構築したいホストのIPアドレスを追記します。
    
    
    CONFIG_NOVA_COMPUTE_HOSTS=192.168.56.20,192.168.56.21

これであとはpackstackコマンドを再度実行するとcomputeノードの構築がssh経由で自動的に行われます。
    
    
    # packstack --answer-file=answer.conf

前回ほどは時間はかかりませんが、そこそこ時間がかかります。インストールが終わったらcomputeノードの/etc/nova/nova.confの下記項目を編集し、ハイパーバイザーをqemuに変更し、再起動をします。
    
    
    libvirt_type=qemu

再起動後にhorizonの管理→システムパネル→ハイパーバイザーを見るとハイパーバイザーが新たに増えてることが確認できます。

[![add_compute_horizon]({{ site.baseurl }}/assets/add_compute_horizon-300x231.png)](http://blog.hirokikana.com/wp-content/uploads/2014/05/add_compute_horizon.png)この状態でインスタンスを作成し「管理→システムパネル→ ハイパーバイザー」から新たに追加したホストをクリックするとcomputeノードでインスタンスが追加されていることがわかります。

[![instance_in_compute_node]({{ site.baseurl }}/assets/instance_in_compute_node-300x134.png)](http://blog.hirokikana.com/wp-content/uploads/2014/05/instance_in_compute_node.png)

また、computeノードの/var/log/nova/compute.logでの追加時のログやtopでのqemu-kvmプロセスが動いていることなどからも新たなcomputeノードにインスタンスが追加できたことがわかると思います。リソースの空き状況によっては既存のOpenStackの方でインスタンスが立ち上がることがありますが、その場合いくつかインスタンスを立ち上げてみることで新しいcomputeノードでも動作が確認できると思います。

## 今後の課題とまとめ

今回はたくさんの仮想マシンを動作させるために必要不可欠なcomputeノードの追加を行いました。packstackを利用することで非常に簡単に追加することができました。

しかし、今後このcomputeノードを運用するにあたり、下記のようなことがわからないと安定した運用はできません。

  * 安全なシャットダウンと復帰の方法
  * 稼働中の仮想マシンのマイグレーション(ライブマイグレーションを含む)



ライブマイグレーションを行う際には共有ストレージの設定や簡単にいかないこともあるようですので、また検証ができたらまとめようと思います。
