---
layout: post
title: Emacs+JDEEでAndroid開発 ~環境構築~
date: 2010-07-15 12:42:46.000000000 +09:00
type: post
published: true
status: publish
categories:
- dev
tags:
- android
- Emacs
- JDEE
meta:
  _edit_last: '1'
  _wp_old_slug: emacsjdee%e3%81%a7android%e9%96%8b%e7%99%ba-%e7%92%b0%e5%a2%83%e6%a7%8b%e7%af%89
author:
  login: admin
  email: hiroki@hirokikana.com
  display_name: hiroki.kana
  first_name: Hiroki
  last_name: Takayasu
---
Android搭載の携帯(HTC Desire)を買ったからには、アプリの開発を行いたいということで今までは情報の多いEclipseを使っていました。  
Eclipse、便利な面も非常に多いのですが私は普段Emacsで全部済ませるタイプなので、いろいろストレスがたまります。

コマンドライン＋Emacsで開発する方法は多く紹介されているので問題ないのですが、ソースコード補完やimportの自動化などについては情報が少なくて見当たらなかったです。  
というわけで、Emacsでソースコード補完やimportの自動化などを行えるような環境構築方法。  
今回使用した環境は、Mac OS X 10.6 Snow Leopard上のCarbon Emacs(22.3.1)です。

  * CEDETのインストール

まず、CEDETをインストールします。  
CEDETはEmacs上でIDEのような機能を実現するためのものらしく、JDEEでも使用しているようです。  
ちなみに、CEDETを使用しない場合はeieio、speedbar、Semantic Bovinatorというのを個別にインストールすればOKなようです。(※未確認)

Carbon EmacsではCEDETをネットワークインストールすることができます。  
Help→Carbon Emacs Package→Net-Install→Cedetをクリックし、インストールします。

[![]({{ site.baseurl }}/assets/cedet-300x180.jpg)](http://blog.hirokikana.com/wp-content/uploads/2010/07/cedet.jpg)

  * JDEEのインストール

sourceforgeのサイト(http://jdee.sourceforge.net/)からJDEEとelibをダウンロード(http://sourceforge.net/projects/jdee/files/)してきます。  
今回はjdee-bin-2.4.0.1とelib-1.0を使用しました。

ダウンロードしてきたら、解凍し適当な場所(私は.emacs.d/lisp以下に置きました)に展開します。  
その際、フォルダ名をjdeとelibに変更しておきました。  
展開したら、.emacsに下記を追加します。
    
        (add-to-list 'load-path "~/.emacs.d/lisp/elib")
    (add-to-list 'load-path "~/.emacs.d/lisp/jdee/lisp")
    
    ;; JDEE
    (load "cedet")
    (setenv "JAVA_HOME" "/System/Library/Frameworks/JavaVM.framework/Versions/CurrentJDK/Home")
    (require 'jde)
    (custom-set-variables
     '(jde-global-classpath (quote (
                                    "/Applications/android-sdk/platforms/android-4/android.jar"
                                    "/Applications/android-sdk/platforms/android-6/android.jar"
                                    "/Applications/android-sdk/platforms/android-7/android.jar"
                                    ))))
    

/Applications/android-sdkはAndroid SDKが配置されているディレクトリです。  
これを追加することでAndroidに関するクラスのソースコード補完などの機能が使えるようになります。

  * jde.elを修正

このままだとtools jarファイルがないって怒られて動作しません。  
「[MacのCarbonEmacsにJDEEを入れる：さむしんぐにゅぅ](http://blog.somethingnew2.com/archives/2007/12/maccarbonemacsj.php)」を参考にjde.elのソースコードを修正します。  
その後、念のため.elcファイルを消去します。(私の環境だと消さないと動作しませんでした)

  * つかってみる

では早速使ってみます。  
例えば下記のようなソースをEmacsで編集します。
    
        package com.hirokikana.android.hello;
    
    public class HelloTest extends Activity
    {
        @Override
        public void onCreate(Bundle savedInstanceState)
        {
            super.onCreate(savedInstanceState);
            setContentView(R.layout.main);
        }
    
        public boolean onCreateOptionsMenu(Menu menu)
        {
            // メニューを追加
            super.onCreateOptionsMenu(menu);
            MenuItem SettingMenu = menu.add(0, 0, 0, "setting");
            SettingMenu.setIcon(android.R.drawable.ic_menu_preferences);
            return true;
        }
    
        public boolean onOptionsItemSelected(MenuItem item)
        {
            switch (item.getItemId()) {
            case 0:
                Intent intent = new Intent();
                intent.setClassName(getPackageName(), getClass.getPackage().getName()+".Setting");
                startActivity(intent);
                return true;
            }
            return false;
        }
    }

この状態で、C-c C-v zを押すと、必要なものがimportされます。  
上記コードの場合、MenuとMenuItemがawtのものとandroidのもので同一の名前があるためどちらをimportするか選択できます。  
[![]({{ site.baseurl }}/assets/import-300x218.png)](http://blog.hirokikana.com/wp-content/uploads/2010/07/import.png)

適当なクラスの後ろでC-c C-v C-.を押すと補完候補の一覧が表示されます。  
[![]({{ site.baseurl }}/assets/hokan-300x124.jpg)](http://blog.hirokikana.com/wp-content/uploads/2010/07/hokan.jpg)  
これは便利なんですが、Carbon Emacsの場合マウスで操作しなきゃいけないのがだいぶめんどいです。  
たぶんターミナルから使うEmacsならキーボードで選択できると思います。

あと、C-c C-v C-lするとprintlnが出ます。  
使ってないですけど(笑)。 


ひとまず、Emacs大好きな私としてはEmacsで自動importが使えてソースコード補完ができれば大満足です。というか最高です。  
さらばEclipse。
