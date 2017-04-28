---
layout: post
title: WordPressでソースコードに色を付けて投稿する
date: 2009-11-08 01:11:53.000000000 +09:00
type: post
published: true
status: publish
categories:
- dev
tags:
- wordpress
meta:
  _edit_last: '1'
  _wp_old_slug: '18'
author:
  login: admin
  email: hiroki@hirokikana.com
  display_name: hiroki.kana
  first_name: Hiroki
  last_name: Takayasu
---
WordPressのプラグインで「Syntax Highlighter and Code Prettifier Plugin for WordPress」というのを入れてソースコードに色づけしてみました。  
余談ですが、WordPressは管理画面からプラグインの検索/インストールができてすごく楽でした。

使い方は下記のようにHTMLを書くだけです。
    
    
    <pre class="brush:[code-alias]"> …Your Code Here </pre>

指定できる言語は[こちら](http://alexgorbatchev.com/wiki/SyntaxHighlighter:Brushes)にまとまっています。

ちなみにPerlではこんな感じになります。
    
    
    #!/usr/bin/perl
    use strict;
    
    print "hogehoge\n";
    
