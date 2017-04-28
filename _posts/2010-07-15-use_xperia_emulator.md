---
layout: post
title: Xperiaのエミュレータを使う
date: 2010-07-15 18:16:12.000000000 +09:00
type: post
published: true
status: publish
categories:
- dev
tags:
- android
meta:
  _edit_last: '1'
  _wp_old_slug: xperia%e3%81%ae%e3%82%a8%e3%83%9f%e3%83%a5%e3%83%ac%e3%83%bc%e3%82%bf%e3%82%92%e4%bd%bf%e3%81%86
author:
  login: admin
  email: hiroki@hirokikana.com
  display_name: hiroki.kana
  first_name: Hiroki
  last_name: Takayasu
---
Xperia(SO-01B)持ってる人結構いますよね。  
町でHTC Desire(X06HT)を持ってる人は全然見かけないですが(とはいえ知り合いで僕以外に2人持ってる人がいますがw)、Xperiaは結構な割合で見かけるんですよね。  
そんなXperiaのアドオンが無償で公開されているということで使ってみることにしました。

まずは[Sony Ericssonの開発者サイト](http://developer.sonyericsson.com/wportal/devworld/)に行きます。  
Downloadsをクリックし、表示されたページのAll Android downloadsをクリックするとXperiaに関するファイルのダウンロードリンクが表示されます。  
その中から「Sony Ericsson Xperia X10 add-on for the Android SDK」をクリックします。  
利用規約などが表示されるので、中身を読みAcceptするとファイルがダウンロードされてきます。

ダウンロードしてきたファイルを展開し、Android SDKのディレクトリ以下のadd-onsフォルダに配置します。  
ターミナルでandroid list targetを実行し、id 5番で"Sony Ericsson Mobile Communications:X10:4"が表示されていれば正しくインストールが完了しています。
    
    
    
    iMac% android list target 
    Available Android targets:
    <略>
    id: 5 or "Sony Ericsson Mobile Communications:X10:4"
         Name: X10
         Type: Add-On
         Vendor: Sony Ericsson Mobile Communications
         Revision: 1
         Description: XPERIA X10 Device
         Based on Android 1.6 (API level 4)
         Skins: WVGA854, X10 (default), HVGA, WVGA800, QVGA
    

次にAVD(Android Virtual Device)を作成します。  
まず、mksdcartコマンドでSDカードイメージを作成します。  
今回は1GBのものを作成しました。
    
    
    
    iMac% mksdcard 1024M xperia_1G_sd.img
    

次にandroidコマンドでAVDを作成します。
    
    
    
    iMac% android create avd --sdcard ~/android/xperia_1G_sd.img --target 5 --name Xperia
    Created AVD 'Xperia' based on X10 (Sony Ericsson Mobile Communications), with the following hardware config:
    hw.dPad=no
    hw.lcd.density=240
    hw.camera.maxHorizontalPixels=3264
    disk.systemPartition.size=144MB
    disk.cachePartition.size=0
    disk.cachePartition=no
    hw.trackBall=no
    hw.camera.maxVerticalPixels=2448
    hw.ramSize=384
    hw.camera=yes
    iMac%        
    

上記コマンドでXperiaという名称でエミュレーターが作成されました。

エミュレータを起動します。
    
    
    
    iMac% emulator @Xperia
    

まさにXperiaです。  
[![]({{ site.baseurl }}/assets/xperia-225x300.png)](http://blog.hirokikana.com/wp-content/uploads/2010/07/xperia.png)  
見た目的な部分でもXperiaにより近い状態なので、おそらく中身もそれに近いとは思います。  
デフォルトでは英語なので、Settings→Locale & text→Select LocaleからJapaneseを選択することで日本語表示になります。  
[![]({{ site.baseurl }}/assets/xperia_jpn-223x300.png)](http://blog.hirokikana.com/wp-content/uploads/2010/07/xperia_jpn.png)

こういうのをメーカーが公開してくれているのは開発者にとってはすごくありがたいですね。  
HTC Desireも出してくれないだろうか。
