---
layout: post
title: Open vSwitchとRyuを使った最も単純なOpenFlowネットワークの構築
date: 2014-12-22 00:42:59.000000000 +09:00
type: post
published: true
status: publish
categories:
- dev
tags:
- OpenFlow
- SDN
meta:
  _edit_last: '1'
  _thumbnail_id: '537'
  _wp_old_slug: open-vswitch%e3%81%a8ryu%e3%82%92%e4%bd%bf%e3%81%a3%e3%81%9f%e6%9c%80%e3%82%82%e5%8d%98%e7%b4%94%e3%81%aaopenflow%e3%83%8d%e3%83%83%e3%83%88%e3%83%af%e3%83%bc%e3%82%af%e3%81%ae%e6%a7%8b%e7%af%89
  _layout: inherit
author:
  login: admin
  email: hiroki@hirokikana.com
  display_name: hiroki.kana
  first_name: Hiroki
  last_name: Takayasu
---
# はじめに

この記事は[ドワンゴ Advent Calendar 2014](http://qiita.com/advent-calendar/2014/dwango)の22日目の記事です。22日目の今回はOpen vSwitchとRyuを使ったOpenFlowネットワークの構築方法を紹介します。  
今回の記事、ドワンゴ Advent Calendarですが、私の現在や過去の業務に密接にOpenFlowが結びついていたわけでもなく、本当に単なる私の趣味によるものであるということを最初に書いておきます。

# OpenFlowとは

SDN(Software Defined Network)という単語を耳にする機会が最近多くなってきたとは感じないでしょうか。SDNというのは今までネットワーク機器が行っていたパケットの経路の制御をソフトウェアで行い、柔軟に制御できるようにすることです。そのソフトウェアはプログラミングすることができ、ネットワークを流れるデータの特性に合わせた制御も可能になります。そのSDNを実現するための技術として注目をされているのがOpenFlowです。

OpenFlowは、ネットワーク機器の制御を行うためのプロトコルでOpen Networking Foundationが中心となり策定を進めています。OpenFlowスイッチ、OpenFlowコントローラー、そしてそれらのやりとりに使われるOpenFlowプロトコルによって構成されています。

  * OpenFlowスイッチ 
    * PCや他のネットワーク機器とつながるスイッチ。一般的なスイッチとは異なり、OpenFlowコントローラーにパケットの制御方法を問い合わせし様々な動作をする。一般的にはハードウェアにより提供されるが、Open vSwitchなどのソフトウェア実装もある。
  * OpenFlowコントローラー 
    * OpenFlowスイッチよりパケットの情報を受け取り、経路を決定するソフトウェア。



OpenFlowの基本的な動きは「OpenFlowスイッチに流れてきたパケット情報をOpenFlowコントローラーに問い合わせ、パケットの内容によって、パケットを制御をする」という流れです。パケットの内容で確認し判定条件として利用できる部分はヘッダフィールドと呼ばれ下記のようなものがあります。

  * スイッチの入力ポート
  * 送信元MACアドレス
  * 宛先MACアドレス
  * Ethernetタイプ
  * VLAN ID
  * VLAN 優先度
  * 送信元IPアドレス
  * 宛先IPアドレス
  * IPプロトコル番号
  * ToSビット
  * 送信元ポート番号
  * 宛先ポート番号



ヘッダフィールドを条件として、下記操作を1つもしくは複数指示することができます。

  * Forward 
    * パケットを指定ポートへ転送する
  * Enqueue 
    * 指定のキューに入れる
  * Drop 
    * パケットを破棄
  * Modify-Field 
    * 指定のフィールドを書き換え



これらの条件判定をパケットが送られてくるごとにされることなく、1度問い合わせた内容はOpenFlowスイッチのフローテーブルという場所に格納されます。フローテーブルに無い条件のパケットのみがOpenFlowコントローラーに問い合わせが行われます。

これがOpenFlowの動きのすべてで、OpenFlowの細かい動きはすべてOpenFlowコントローラー内で動作するプログラムで定義することができます。

# OpenFlowを利用したネットワークを構築するには

必要になるのはOpenFlowコントローラーとOpenFlowスイッチです。OpenFlowコントローラーはオープンソースで提供されているものもありますが、OpenFlowスイッチはハードウェアで提供されているもの(NEC UNIVERGE PFシリーズなど)を揃えようとすると手軽に手を出せる価格のものはありませんし、個人で購入するのも難しいでしょう。

そこでOpenFlowスイッチをソフトウェアとして実装されているものを利用するのが現実的です。今回はOpenFlowスイッチの代表的な実装のOpen vSwitch、また、OpenFlowコントローラーはPythonでコントローラーを実装することができるRyuというフレームワークを利用しOpenFlowネットワークを構築する方法を紹介します。

# 今回行うこと

今回はOpenFlowスイッチとなるマシンとOpenFlowコントローラーを同一のマシンに用意します。さらに仮想マシンだけがアクセスできる内部ネットワークに、内部ネットワークのみにつながっているマシンを用意し、ホストマシンから内部ネットワークのみにつながっているマシンに対してOpenFlowスイッチを経由してアクセスできるようにします。それぞれ仮想マシンはCentOS 6.5、ホストマシンはMac OS X 10.10を利用しました。

[![dwango_adventcalendar2014_1]({{ site.baseurl }}/assets/dwango_adventcalendar2014_1.jpg)](http://blog.hirokikana.com/wp-content/uploads/2014/12/dwango_adventcalendar2014_1.jpg)

# 各マシンの準備

OpenFlowコントローラーおよびOpenFlowスイッチの役割をするマシンを以後ofnw01と呼び、内部ネットワーク内の仮想マシンをinternal01と呼びます。構成ですが、2台のマシンに共通している部分は下記のとおりです。

  * OS 
    * CentOS 6.6 x86_64 minimal
  * メモリ 
    * 1GB
  * HDD 
    * 20GB
  * CPUコア数 
    * 1



2台のマシンで異なるのはNICの構成です。

  * ofnw01 
    * NIC1 : ホストオンリーネットワーク、プロミスキャスモードをすべて許可
    * NIC2：内部ネットワーク、プロミスキャスモードをすべて許可
    * NIC3：NAT(インターネットに接続するため)
  * internal01 
    * NIC1：内部ネットワーク
    * NIC2：NAT(インターネットに接続するため)



NATのNICについてはOSインストール後にDHCPでIPアドレスを取得するようにし、インターネットに接続できるようにしておきます。2つのマシンが用意できたら下記コマンドを実行し、OSを最新の状態にアップデートしておきます。
    
    
    # yum update
    # reboot

# OpenFlowスイッチの準備

ofnw01にまずOpenFlowスイッチとして動作するOpen vSwitchをインストールします。今回は最も手軽に導入することができる方法としてRedHat系OS向けのOpenStackリポジトリであるRDOを利用します。
    
    
    # yum install http://rdo.fedorapeople.org/openstack-icehouse/rdo-release-icehouse.rpm
    # yum  install openvswitch
    # service openvswitch start
    # chkconfig openvswitch on

上記のコマンドを実行するだけで導入することができ、非常に手軽です。執筆時点では2.1.3がインストールされました。
    
    
    #  ovs-vsctl show
    6d7c9a72-02a7-41cb-abb8-f11bccad3e62
    ovs_version: 
    "2.1.3"

次にOpen vSwitchでブリッジの設定を行います。このとき内部ネットワークとホストオンリーネットワークで利用するNICにはIPアドレスが振られていないことを確認したうえで作業を行います。下記コマンドでbr0ブリッジを新規作成し、eth0とeth1をbr0ブリッジにポートとして登録します。
    
    
    # ovs-vsctl add-br br0
    # ovs-vsctl add-port br0 eth0
    # ovs-vsctl add-port br0 eth1

図にすると下記のようなイメージになります。

[![dwango_adventcalendar2014_2]({{ site.baseurl }}/assets/dwango_adventcalendar2014_2.jpg)](http://blog.hirokikana.com/wp-content/uploads/2014/12/dwango_adventcalendar2014_2.jpg)

この状態でbr0はスイッチングハブとして動作していますので、内部ネットワークとホストオンリーネットワークは疎通している状態になっています。それを確認するために、internal01のeth0に対してホストオンリーネットワーク内で利用できるIPアドレスを付与し、ホストマシンと疎通が取れるか確認します。

**internal01の/etc/sysconfig/network-scripts/ifcfg-eth0**
```
DEVICE=eth0  
TYPE=Ethernet  
ONBOOT=yes  
NM_CONTROLLED=yes  
BOOTPROTO=static  
HWADDR=08:00:27:40:52:0D  
IPADDR=172.16.0.200  
NETMASK=255.255.255.0
```

上記設定を確認し、ホストマシンからinternal01へpingをしてみます。
    
    
    $ ping 172.16.0.200
    PING 172.16.0.200 (172.16.0.200): 56 data bytes
    64 bytes from 172.16.0.200: icmp_seq=0 ttl=64 time=2.424 ms
    64 bytes from 172.16.0.200: icmp_seq=1 ttl=64 time=2.436 ms
    64 bytes from 172.16.0.200: icmp_seq=2 ttl=64 time=3.052 ms
    64 bytes from 172.16.0.200: icmp_seq=3 ttl=64 time=0.877 ms

試しにofnw01のopenvswitchを止めてみると疎通しなくなるのがわかると思います。

これでOpenFlowスイッチとして利用するブリッジの準備は完了です。

# OpenFlowコントローラーの準備

次にOpenFlowコントローラーを準備します。今回はOpenFlowスイッチを構築したofnw01上でOpenFlowコントローラーを動作させます。今回利用するOpenFlowコントローラーはRyu(<http://osrg.github.io/ryu/>)を利用します。RyuはPythonでOpenFlowコントローラーを実装することができるフレームワークです。まずはRyuを利用するためにPythonのパッケージマネージャーであるpipのインストール後にRyuのインストールを行います。

まずpipを利用するためにepelリポジトリを有効にします。
    
    
    # yum install epel-release

次にpipとRyuのビルドに必要なものをインストールし、最後にpipを使いRyuをインストールします。
    
    
    # yum install python-pip python-devel gcc gcc-c++
    # pip install ryu
    # pip install --upgrade netaddr

これでRyuを利用する環境の構築は完了です。

次にRyuでスイッチングハブの役割およびトラフィックモニターを行うこちら(<http://osrg.github.io/ryu-book/ja/html/traffic_monitor.html>)のサンプルを利用します。このサンプルをほとんど利用しているのですが、少し修正をいれてOpenFlowスイッチがOpenFlowコントローラーに接続したときにログ出力を行うようにしたのが下記のコードです。
    
```python    
from operator import attrgetter
from ryu.app import simple_switch_13
from ryu.controller import ofp_event
from ryu.controller.handler import MAIN_DISPATCHER, DEAD_DISPATCHER, CONFIG_DISPATCHER
from ryu.controller.handler import set_ev_cls
from ryu.lib import hub
 
class SimpleMonitor(simple_switch_13.SimpleSwitch13):
    def __init__(self, *args, **kwargs):
        super(SimpleMonitor, self).__init__(*args, **kwargs)
        self.datapaths = {}
        self.monitor_thread = hub.spawn(self._monitor)
 
    @set_ev_cls(ofp_event.EventOFPSwitchFeatures, CONFIG_DISPATCHER)
        def switch_features_handler(self, ev):
        self.logger.info('switch joind: datapath: %016x' % ev.msg.datapath.id)
 
    @set_ev_cls(ofp_event.EventOFPStateChange,
                [MAIN_DISPATCHER, DEAD_DISPATCHER])
    def _state_change_handler(self, ev):
        datapath = ev.datapath
        if ev.state == MAIN_DISPATCHER:
            if not datapath.id in self.datapaths:
                self.logger.debug('register datapath: %016x', datapath.id)
                self.datapaths[datapath.id] = datapath
        elif ev.state == DEAD_DISPATCHER:
            if datapath.id in self.datapaths:
                self.logger.debug('unregister datapath: %016x', datapath.id)
                del self.datapaths[datapath.id]
 
 
    def _monitor(self):
        while True:
            for dp in self.datapaths.values():
                self._request_stats(dp)
            hub.sleep(10)
 
 
    def _request_stats(self, datapath):
        self.logger.debug('send stats request: %016x', datapath.id)
        ofproto = datapath.ofproto
        parser = datapath.ofproto_parser
        req = parser.OFPFlowStatsRequest(datapath)
        datapath.send_msg(req)
        req = parser.OFPPortStatsRequest(datapath, 0, ofproto.OFPP_ANY)
        datapath.send_msg(req)
 
 
    @set_ev_cls(ofp_event.EventOFPFlowStatsReply, MAIN_DISPATCHER)
    def _flow_stats_reply_handler(self, ev):
        body = ev.msg.body
        self.logger.info('datapath         '
                         'in-port  eth-dst           '
                         'out-port packets  bytes')
        self.logger.info('
```