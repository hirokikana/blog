---
layout: post
title: WordPressのエディタでFiremacsをちゃんと使えるようにする
date: 2009-11-08 12:47:22.000000000 +09:00
type: post
published: true
status: publish
categories:
- dev
tags:
- wordpress
meta:
  _edit_last: '1'
  _wp_old_slug: '32'
author:
  login: admin
  email: hiroki@hirokikana.com
  display_name: hiroki.kana
  first_name: Hiroki
  last_name: Takayasu
---
ブラウザは常々Firefoxを使っていて、Firemacsは今やなくてはならないエクステンションになってます、私の中では。

WordPressの投稿に使用するエディタでCtrl+iやらその他Emacsのキーバインドを使うとWordPressのキーバインドとぶつかってしまい、思いもよらない動きに…orz  
どうにかならんかと調べてみたら、Firefoxの設定(about:config)でui.key.generalAccessKey = 0にすれば万事OKらしい。

これで一安心。  
というか最新のfiremacsを入れたらCtrl-mで改行しなくて非常に不便なんだが…  
会社のWindowsではきちんと動くからSnow Leopard上のFirefox特有の問題?

**参考**

  * WordPressの投稿画面のアクセスキーがMacのemacsキーバインドとバッティングするのを無効化　(<http://shokai.org/blog/archives/2359>)


