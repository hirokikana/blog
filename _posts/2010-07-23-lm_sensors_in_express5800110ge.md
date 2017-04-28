---
layout: post
title: Express5800/110Ge上のCentOSでlm_sensorsを使う
date: 2010-07-23 01:08:57.000000000 +09:00
type: post
published: true
status: publish
categories:
- dev
tags:
- lm_sensors
meta:
  _edit_last: '1'
  _wp_old_slug: express5800110ge%e4%b8%8a%e3%81%aecentos%e3%81%a7lm_sensors%e3%82%92%e4%bd%bf%e3%81%86
author:
  login: admin
  email: hiroki@hirokikana.com
  display_name: hiroki.kana
  first_name: Hiroki
  last_name: Takayasu
---
Express5800/110Geでlm_sensorsを使用してみました。

この機種はIT8718Fを使用していますが、lm_sensorsのサイトで確認すると2.6.19以降で対応しているようです。  
CentOS 5は2.6.18なので対応していないと思いきや、調べてみると2.6.18-170.el5以降で対応しているとのことでした。

さっそく、lm_sensorsをインストールしました。
    
    
    
    [root@minori ~]# yum install lm_sensors
    Loaded plugins: fastestmirror
    Loading mirror speeds from cached hostfile
     * addons: www.ftp.ne.jp
     * base: www.ftp.ne.jp
     * extras: www.ftp.ne.jp
     * updates: www.ftp.ne.jp
    Setting up Install Process
    Resolving Dependencies
    --> Running transaction check
    
