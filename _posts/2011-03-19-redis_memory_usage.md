---
layout: post
title: Redisとデータ構造によるメモリ消費量の違い
date: 2011-03-19 15:21:43.000000000 +09:00
type: post
published: true
status: publish
categories:
- dev
tags:
- Python
- Redis
meta:
  _edit_last: '1'
  _wp_old_slug: redis%e3%81%a8%e3%83%87%e3%83%bc%e3%82%bf%e6%a7%8b%e9%80%a0%e3%81%ab%e3%82%88%e3%82%8b%e3%83%a1%e3%83%a2%e3%83%aa%e6%b6%88%e8%b2%bb%e9%87%8f%e3%81%ae%e9%81%95%e3%81%84
author:
  login: admin
  email: hiroki@hirokikana.com
  display_name: hiroki.kana
  first_name: Hiroki
  last_name: Takayasu
---
今回はRedisというKVSのちょっとした紹介と、利用しているうえでちょっと気になったことがあったので、簡単ですがまとめました。  
最初に書いておきますが、話があんまりまとまってません。

RedisとはNoSQLと呼ばれるKVS(Key Value Store)のひとつです。  
他のKVSと異なる特長はある程度の永続化が可能ということと、多くのデータ構造を使用できるという点です。  
多くのオンメモリのKVSの場合電源断などでデータが飛んでしまいますが、Redisは一定のタイミング(設定で随時書き込みにもできます)でデータをディスクにバックアップするためそのような心配は少ないです。  
また、値として配列やハッシュのような構造を格納できるとともに、配列同士の集合演算も行うことができます。

Redisですが、最近仕事で触れる機会があったり、ちょっとしたことにでも利用したりしています。  
ちょっとしたアプリのプロトタイプを作る際にRDBMSのスキーマって結構変わったりすると思います。(私の最初の設計が悪いのですが…)  
テーブルのスキーマを途中で変えたりするのは意外と面倒だったりします。  
そんなときに、データの格納場所としてRDBMSの代わりにRedisを使用し、ある程度データの格納方法が固まったところで必要に応じてRDBMSに移行するということで表側をサクサクと作ることができます。  
(うまいこといったらRDBMS使わないでそのままRedisを使えば良いわけですし)

さて、今回はそんなRedisでキーを大量に追加したところ思った以上にメモリの容量を食ったので、ちょっとした測定をしてみました。

たとえばユーザIDからユーザ名を引くのをRedisを使ってやりたいとします。  
１つ目の解決策はまずキーにユーザID、値にユーザ名を入れるという風にしました。  
違うアプローチとしてハッシュ型を使用して、usernameというキーでハッシュキーとしてユーザID、値としてユーザ名を入れるようにします。  
この際の２つの使用メモリ量を比較してみようと思います。

比較を行うために下のようなPythonスクリプトを書きました。
    
    
    
    #!/usr/bin/env python
    # -*- coding:utf-8 -*-
    
    from redis import Redis
    from time import time
    
    def printInfo(redis_con):
        info = redis_con.info()
        print "
