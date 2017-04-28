---
layout: post
title: さくらVPS移行記 〜Webサーバ設定編〜
date: 2011-08-13 10:58:56.000000000 +09:00
type: post
published: true
status: publish
categories:
- dev
tags:
- Apache
- Linux
- MySQL
- さくらVPS
meta:
  _edit_last: '1'
  _wp_old_slug: "%e3%81%95%e3%81%8f%e3%82%89vps%e7%a7%bb%e8%a1%8c%e8%a8%98-%e3%80%9cweb%e3%82%b5%e3%83%bc%e3%83%90%e8%a8%ad%e5%ae%9a%e7%b7%a8%e3%80%9c"
author:
  login: admin
  email: hiroki@hirokikana.com
  display_name: hiroki.kana
  first_name: Hiroki
  last_name: Takayasu
---
さて、間が空いてしまいました。  
さくらVPSに移行してから、かれこれ数ヶ月経ちました。  
512MBしかないメモリに不安を抱いていましたが、特に問題なく運用できています。  
今回はブログの移行に伴って設定したWebサーバ周りの話をしたいと思います。

まず、Apache、PHPとMySQLをインストールしないことには始まりません。
    
    
    
    $ sudo yum install httpd
    $ sudo yum install php53 php53-mysql
    $ sudo yum install mysql-server
    

まずApacheから設定していきます。  
さくらVPSはあまり潤沢なメモリがあるわけではないので、メモリを節約するためにいらないモジュール群をコメントアウトしてロードしないようにします。  
私の環境では下記のモジュール群を/etc/httpd/conf/httpd.confと/etc/httpd/conf.d/proxy_ajp.confの以下の行をコメントアウトしました。
    
    
    
    #LoadModule ldap_module modules/mod_ldap.so
    #LoadModule authnz_ldap_module modules/mod_authnz_ldap.so
    #LoadModule dav_module modules/mod_dav.so
    #LoadModule dav_fs_module modules/mod_dav_fs.so
    #LoadModule rewrite_module modules/mod_rewrite.so
    #LoadModule proxy_module modules/mod_proxy.so
    #LoadModule proxy_balancer_module modules/mod_proxy_balancer.so
    #LoadModule proxy_ftp_module modules/mod_proxy_ftp.so
    #LoadModule proxy_http_module modules/mod_proxy_http.so
    #LoadModule proxy_connect_module modules/mod_proxy_connect.so
    #LoadModule cern_meta_module modules/mod_cern_meta.so
    #LoadModule asis_module modules/mod_asis.so
    

**/etc/httpd/conf.d/httpd.conf**
    
    
    
    #LoadModule proxy_ajp_module modules/mod_proxy_ajp.so
    

**/etc/httpd/conf.d/proxy_ajp.conf**

さらに、httpd.confで起動するプロセス数などを調整しました。  
あまり私のブログには残念ながら訪問者が少ないために、控えめな設定にしました。  
今のところこの設定で詰まってるとかいうことは（たぶん）ないと思います。  
（そもそもこの設定だと詰まるくらいに訪問者を増やすための努ry…）
    
    
    
    <IfModule prefork.c>
    StartServers       3
    MinSpareServers    3
    MaxSpareServers    5
    ServerLimit       15
    MaxClients        15
    MaxRequestsPerChild  4000
    </IfModule>
    

さて、ここまででApacheの設定は完了です。  
次にMySQLの設定を行います。

まずMySQLを起動します。
    
    
    
    $ sudo /etc/init.d/mysqld start
    

次に初期設定を行うためのスクリプトが用意されているので、それを実行します。  
このスクリプトではMySQLのrootユーザーにパスワードを設定、匿名ユーザの削除、testデーターベースの削除などを質問に答えるだけでやってくれます。
    
    
    
    $ sudo mysql_secure_installation
    NOTE: RUNNING ALL PARTS OF THIS SCRIPT IS RECOMMENDED FOR ALL MySQL
          SERVERS IN PRODUCTION USE!  PLEASE READ EACH STEP CAREFULLY!
    
    
    In order to log into MySQL to secure it, we'll need the current
    password for the root user.  If you've just installed MySQL, and
    you haven't set the root password yet, the password will be blank,
    so you should just press enter here.
    
    Enter current password for root (enter for none):
    OK, successfully used password, moving on...
    
    Setting the root password ensures that nobody can log into the MySQL
    root user without the proper authorisation.
    
    You already have a root password set, so you can safely answer 'n'.
    
    Change the root password? [Y/n] <y>
    New password: <パスワード>
    Re-enter new password: <パスワード>
    Password updated successfully!
    Reloading privilege tables..
     ... Success!
    
    
    By default, a MySQL installation has an anonymous user, allowing anyone
    to log into MySQL without having to have a user account created for
    them.  This is intended only for testing, and to make the installation
    go a bit smoother.  You should remove them before moving into a
    production environment.
    
    Remove anonymous users? [Y/n] <y>
     ... Success!
    
    Normally, root should only be allowed to connect from 'localhost'.  This
    ensures that someone cannot guess at the root password from the network.
    
    Disallow root login remotely? [Y/n] <y>
     ... Success!
    
    By default, MySQL comes with a database named 'test' that anyone can
    access.  This is also intended only for testing, and should be removed
    before moving into a production environment.
    
    Remove test database and access to it? [Y/n] <y>
     - Dropping test database...
     ... Success!
     - Removing privileges on test database...
     ... Success!
    
    Reloading the privilege tables will ensure that all changes made so far
    will take effect immediately.
    
    Reload privilege tables now? [Y/n] <y>
     ... Success!
    
    Cleaning up...
    
    
    
    All done!  If you've completed all of the above steps, your MySQL
    installation should now be secure.
    
    Thanks for using MySQL!
    

これで匿名ユーザの削除、rootユーザのパスワード設定、rootユーザーのリモートログイン不可、testデータベースの削除が行えました。  
次に普段使用するユーザを追加します。
    
    
    
    % mysql -u root -p
    Enter password: <パスワード>
    Welcome to the MySQL monitor.  Commands end with ; or \g.
    Your MySQL connection id is 18641
    Server version: 5.0.77 Source distribution
    
    Type 'help;' or '\h' for help. Type '\c' to clear the buffer.
    
    mysql&g; GRANT ALL ON *.* TO <ユーザ名>@"localhost" IDENTIFIED BY "<パスワード>";
    Query OK, 0 rows affected (0.00 sec)
    
    mysql> FLUSH PRIVILEGES;
    

これでMySQLの設定も完了です。

次に今まで使っていたWordPressを新環境に移行します。  
まず旧環境のDBをdumpします。
    
    
    
    $ mysqldump -p <DB名> > db.dump
    

新環境にSCPなりでdumpしたファイルをコピーし、新DBにデータを挿入します。
    
    
    
    $ mysql -u <ユーザ名> -p < db.dump
    

次に旧環境のWordPressディレクトリを新環境にコピーしてきます。  
新環境でWordPressを新たにインストールして…と思ったのですが、ディレクトリ下に画像やらプラグインやらのデータとかもあったので、そのままコピーすることにしました。  
wp-config.phpのDBの値を新環境のものとあわせて設定完了です。
    
    
    
    define('DB_NAME', '<DB名>');
    define('DB_USER', '<DBのユーザ名>');
    define('DB_PASSWORD', '<パスワード>');
    define('DB_HOST', 'localhost');
    

このブログはサーバー内でvirtualhostを切っているのでApacheの設定に下記を追加しました。
    
    
    
    NameVirtualHost *:80
    <VirtualHost *:80>
         ServerName blog.hirokikana.com
         DocumentRoot /path/to/blogdata_dir
    </VirtualHost>
    

最後にApacheを起動して正しくブログにアクセスできるか確認しました。  
これで一通り移行ができました。

1ヶ月程度この状態で使っていますが、今のところ特に不便はしていません。  
また、メモリも半分程度残っており、心配したほど逼迫することは今のところありません。  
毎月1,000円程度で固定IPとちょっとしたLinux環境が手に入ると思えば結構お得だと思います。
