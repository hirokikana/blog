---
layout: post
title: LVMを使用しインストールしたDomain Uのディスク拡張
date: 2009-11-15 16:43:32.000000000 +09:00
type: post
published: true
status: publish
categories:
- dev
tags:
- Xen
meta:
  _edit_last: '1'
  _wp_old_slug: lvm%e3%82%92%e4%bd%bf%e7%94%a8%e3%81%97%e3%82%a4%e3%83%b3%e3%82%b9%e3%83%88%e3%83%bc%e3%83%ab%e3%81%97%e3%81%9fdomain-u%e3%81%ae%e3%83%87%e3%82%a3%e3%82%b9%e3%82%af%e6%8b%a1%e5%bc%b5
author:
  login: admin
  email: hiroki@hirokikana.com
  display_name: hiroki.kana
  first_name: Hiroki
  last_name: Takayasu
---
我が家の回線はADSL 8Mと非常に貧弱なので、yum updateなどが非常に遅いです。  
さらにyum updateをやっている間はインターネットへの疎通全体が遅くなるというなんとも微妙な感じでした。  
なので、ローカルにCentOSのミラーサーバーを構築しようと思い、Xen上に構築してある自宅内の雑多サービスサーバ(DNSやLDAPなどなど)にrsyncをしていたらDisk fullに…orz  
20GBのディスクイメージだとちょっと足りなかったようです。

再インストールするというのも非常に面倒なのでXenのディスクイメージを拡張してみました。

まず、[ここ](http://project-p.jp/halt/anubis/blog_show/986)を参考に以下の手順でやってみたんですが、うまくいかず。

  * DomainUをシャットダウン

普通にshutdown -h nowコマンドを使用しました。

  * ddで拡張したい分の空のディスクイメージを作成する

今回は12G拡張したいので12GBのイメージを作成しました。
    
        # dd if=/dev/zero of=12GB.img bs=1024k count=12288

  * catコマンドで既存のディスクイメージと結合

なんてことはなくcatでくっつけるだけです。haruka.imgというのが古いイメージです。
    
        # mv haruka.img haruka.img.old
    # cat haruka.img.old 12GB.img > haruka.img

  * e2fsck -vfでディスクイメージをチェック

ここで下記のエラーが出て先に進めませんでした。
    
        e2fsck -vf haruka.img
    e2fsck 1.39 (29-May-2006)
    Couldn't find ext2 superblock, trying backup blocks...
    e2fsck: Bad magic number in super-block while trying to open haruka.img
    
    The superblock could not be read or does not describe a correct ext2
    filesystem.  If the device is valid and it really contains an ext2
    filesystem (and not swap or ufs or something else), then the superblock
    is corrupt, and you might try running e2fsck with an alternate superblock:
        e2fsck -b 8193 
    

指示のとおりにe2fsck -b 8193 -vf haruka.imgとやってみたも変わらず。 


ここでだいぶはまりました。  
原因はDomainUのOSをLVMで導入したためでした。  
最近はほとんどデフォルトだとLVMで導入されますよね。  
LVMでなければ、外部からe2fsckなどのコマンドを使用しイメージを操作できるのですが、LVMの場合はその仕組み上違った方法を使用する必要があります。  
下記の手順でうまくいきました。ちなみに上記手順のcatでイメージをつけるところまでは同じなので、その続きから書きます。

  1. losetupコマンドでディスクイメージをループバックマウントする

losetupコマンドを使用してループバックマウントします。  
losetupなんてコマンドがあるんですね、初めて知りました。
    
        # losetup -f
    /dev/loop1
    # losetup /dev/loop1 haruka.img

  2. fdiskコマンドで領域の拡張を行う

一度、LVMパーティションの削除を行って再設定を行います。  
この辺のログを取りのがしたので参考サイトのをコピペさせてもらいます。
    
        # fdisk  /dev/loop1
    Command (m for help): d
    Partition number (1-4): 2
    
    Command (m for help): n
    Command action
       e   extended
       p   primary partition (1-4)
    p
    Partition number (1-4): 2
    First cylinder (14-652, default 14):
    Using default value 14
    Last cylinder or +size or +sizeM or +sizeK (14-652, default 652):
    Using default value 652
    
    Command (m for help): t
    Partition number (1-4): 2
    Hex code (type L to list codes): 8e
    Changed system type of partition 2 to 8e (Linux LVM)
    
    Command (m for help): p
    
    Disk /dev/loop1: 5368 MB, 5368709120 bytes
    255 heads, 63 sectors/track, 652 cylinders
    Units = cylinders of 16065 * 512 = 8225280 bytes
    
          Device Boot      Start         End      Blocks   Id  System
    /dev/loop1p1   *           1          13      104391   83  Linux
    /dev/loop1p2              14         652     5132767+  8e  Linux LVM
    
    Command (m for help): w
    The partition table has been altered!
    
    Calling ioctl() to re-read partition table.
    
    WARNING: Re-reading the partition table failed with error 22: Invalid argument.
    The kernel still uses the old table.
    The new table will be used at the next reboot.
    Syncing disks.
    

最後にループバックでバイスをアンマウントします。
    
        losetup -d /dev/loop1

最後にDomainUを起動しておきましょう。
    
        xm create haruka

  3. PVのリサイズとLVの拡張

DomainUが起動したらPVを現在のデバイスのサイズに合わせて拡張し、LVの拡張も行います。  
いろいろログとり忘れてしまったのでコマンドだけ。
    
        # pvresize /dev/xvda2
    # lvresize -L +12G /dev/VolGroup00/LogVol00




これで無事ディスク領域の拡張ができました。

**参考サイト**

  * [harry’s memorandum Xenのディスクイメージファイルを拡張してみた](http://d.hatena.ne.jp/dharry/20090416/1239822866)
  * [xenのディスクイメージを拡張する - /halt/Snapshot](http://project-p.jp/halt/anubis/blog_show/986)
  * [Tomo's page イメージファイル（HDD）の拡張（リサイズ）する方法](http://good-stream.com/goodstream/xen/centos5/imgfile.html)


