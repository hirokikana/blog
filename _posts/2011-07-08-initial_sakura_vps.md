---
layout: post
title: さくらVPS移行記 〜初期設定〜
date: 2011-07-08 00:10:25.000000000 +09:00
type: post
published: true
status: publish
categories:
- dev
tags:
- CentOS
- Linux
- さくらVPS
meta:
  _edit_last: '1'
  _wp_old_slug: "%e3%81%95%e3%81%8f%e3%82%89vps%e7%a7%bb%e8%a1%8c%e8%a8%98"
author:
  login: admin
  email: hiroki@hirokikana.com
  display_name: hiroki.kana
  first_name: Hiroki
  last_name: Takayasu
---
かれこれ数年、実家にサーバーを設置していたのですが、最近ではあまりリソースも消費せず電気代ばかりかかっているような状況でした。  
さらに昨今の電力状況を鑑みてもできればあまり無駄に電気は消費しない方が良いのではないだろうかという気持ちになっていました。

そこで、最近ではすっかり安くなったVPSを使ってみることにしました。  
どこを使おうか迷った末、比較的安価かつ回線が速そうで評判も悪くなさそうな「さくらのVPS 512」を利用することにしました。  
ちなみに引っ越すのは取り急ぎこのブログだけ引っ越しました。  
(すでにこのブログはさくらのVPS上で動かしています)  
引っ越しに際してやった作業を備忘録的に残しておこうと思います。

ちなみにさくらのVPSは申し込み後間もなくメールが届いて利用できるようになりました。  
何度か初期化を行ったりしてみたのですが、非常にスムーズに初期化も行えてすばらしい限りです。  
申し込んですぐに固定IPとLinux環境が手に入るなんて、ホントいい時代になったものです。

### ユーザーの作成とSSHの設定

まずは普段利用するユーザーの追加を行います。
    
    
    
    # useradd -g users hiroki
    # passwd hiroki
    

次にSSH経由でのrootのログインを無効にするとともに、ポート番号を22から任意の番号に変更しました。  
rootユーザーでSSH経由でログインすることはほとんどないだろうというのと、rootユーザーでブルートフォースアタックをかけてくる輩がいるので、その対策です。  
加えてポート番号の変更はスタンダードな22番よりほかのポート番号の方が同じく変な輩にアタックされないだろうとのところから行います。  
まずは「/etc/ssh/sshd_config」の項目を編集します。
    
    
    
    Port XXXX(設定したいポート番号)
    PermitRootLogin no
    

編集が終わったらSSHサーバーを再起動します。
    
    
    
    # /etc/init.d/sshd restart
    

さらに今後作業の際には一般ユーザーでsudoを使って作業するようにします。  
sudoを使うとrootのパスワードを打つ必要がなくなるとともに、/var/log/secureにログが残るのでなんとなく安心です。  
visudoコマンドで設定ファイルを書き換えることができますので、自分に対してsudoを使う権限を足しておきます。
    
    
    
    hiroki ALL=(ALL)   ALL
    

これでSSH経由で作業をする準備が整いました。ここまではさくらの管理画面などからシリアルポート経由で作業すると良いでしょう。

### ファイアウォールの設定

今回はひとまずSSHとHTTP以外を取り急ぎ使わないので他を閉じておくというポリシーにします。  
「/etc/sysconfig/iptable」ファイルを作成し、下記のような内容を記述します。
    
    
    
    *filter
    :INPUT          ACCEPT  [0:0]
    :FORWARD        ACCEPT  [0:0]
    :OUTPUT         ACCEPT  [0:0]
    :FW-1-INPUT     -       [0:0]
    
    -A INPUT -j FW-1-INPUT
    -A FORWARD -j FW-1-INPUT
    -A FW-1-INPUT -i lo -j ACCEPT
    -A FW-1-INPUT -p icmp --icmp-type any -j ACCEPT
    -A FW-1-INPUT -m state --state ESTABLISHED,RELATED -j ACCEPT
    
    -A FW-1-INPUT -m state --state NEW -m tcp -p tcp --dport XXXX -j ACCEPT
    -A FW-1-INPUT -m state --state NEW -m tcp -p tcp --dport 80 -j ACCEPT
    
    -A FW-1-INPUT -j REJECT --reject-with icmp-host-prohibited
    
    COMMIT
    

さらに/etc/sysconfig/iptables-configの下記項目も編集しておきます。
    
    
    
    IPTABLES_SAVE_COUNTER="yes"
    

最後にiptablesを再起動します。
    
    
    
    $ sudo /etc/init.d/iptables restart
    

先ほど設定したポリシーが反映せれているかはiptablesコマンドで確認することができます。  
実際にポートが閉じてるかなどは、適当なポートで待ち受けるサービスを立ち上げて確認すると良いと思います。
    
    
    
    $ sudo /sbin/iptables -L
    Chain INPUT (policy ACCEPT)
    target     prot opt source               destination         
    FW-1-INPUT  all  --  anywhere             anywhere            
    
    Chain FORWARD (policy ACCEPT)
    target     prot opt source               destination         
    FW-1-INPUT  all  --  anywhere             anywhere            
    
    Chain OUTPUT (policy ACCEPT)
    target     prot opt source               destination         
    
    Chain FW-1-INPUT (2 references)
    target     prot opt source               destination         
    ACCEPT     all  --  anywhere             anywhere            
    ACCEPT     icmp --  anywhere             anywhere            icmp any 
    ACCEPT     all  --  anywhere             anywhere            state RELATED,ESTABLISHED 
    ACCEPT     tcp  --  anywhere             anywhere            state NEW tcp dpt:XXXX 
    ACCEPT     tcp  --  anywhere             anywhere            state NEW tcp dpt:http 
    REJECT     all  --  anywhere             anywhere            reject-with icmp-host-prohibited 
    

### アップデート&その他調整

次に各種パッケージをアップデートしておきましょう。
    
    
    
    $ sudo yum update
    

さくらのVPSではいろいろな無駄と思われるサービスはあらかじめoffにされていました。  
なので、あまり手間をかけて止める必要はなかったのですが、iSCSI系のサービスが動いていたので、それだけ止めました。
    
    
    
    $ sudo /etc/init.d/iscsid stop
    $ sudo /etc/init.d/iscsi stop
    $ sudo /sbin/chkconfig iscsid off
    $ sudo /sbin/chkconfig iscsi off
    

最後にホスト名の設定をするために「/etc/sysconfig/network」の下記の設定を編集します。
    
    
    
    HOSTNAME=<設定したいホスト名>
    

ここまで終わったら再起動をします。

ひとまず今回はとりあえず使えるようにするための初期設定を行いました。  
続きのApache/MySQL/WordPressの移行については、また次の機会に書きます。
