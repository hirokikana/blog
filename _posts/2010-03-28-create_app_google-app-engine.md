---
layout: post
title: Google App Engineを入門してみよう　〜アプリ作成編〜
date: 2010-03-28 17:12:48.000000000 +09:00
type: post
published: true
status: publish
categories:
- dev
tags:
- GoogleAppEngine
meta:
  _edit_last: '1'
  _wp_old_slug: google-app-engine%e3%82%92%e5%85%a5%e9%96%80%e3%81%97%e3%81%a6%e3%81%bf%e3%82%88%e3%81%86%e3%80%80%e3%80%9c%e3%82%a2%e3%83%97%e3%83%aa%e4%bd%9c%e6%88%90%e7%b7%a8%e3%80%9c
author:
  login: admin
  email: hiroki@hirokikana.com
  display_name: hiroki.kana
  first_name: Hiroki
  last_name: Takayasu
---
前回([Google App Engineを入門してみよう](http://blog.hirokikana.com/?p=119))に続き、練習に簡単なアプリを作ってみました。

ひとまずドキュメントに載ってたサンプルプログラムを少し改造すればできそうだという理由で、Twitterのようなものを作ってみました。  
[「みななう 〜Mina-Now〜」](http://mina-now.appspot.com)  
[![]({{ site.baseurl }}/assets/minanow-300x214.png)](http://blog.hirokikana.com/wp-content/uploads/2010/03/minanow.png)

ログインを押すとGoogleのログイン画面が出るので、Googleのアカウントを入れてログインします。  
こういう認証のためのコードをいっさい書かなくてよいのがGoogle App Engineは便利ですね。  
みななうではアカウントのニックネームをユーザ名として利用しています。

ユーザごとにアイコンも設定することができます。  
アイコンは現状ではjpgのみ対応(とはいえバリデートしてません)なので、将来的にはいろんなフォーマットに対応させようかと思います。  
あとは画像処理のAPIがGoogle App Engineにあるらしいので、将来的にはアップロードのときのリサイズとかに使ってみます。

こういうアプリが簡単につくれてしまうのが、Google App Engineの良いところですね。  
ソースは早速[github](http://github.com/hirokikana/MinaNow)に上げておきました。
