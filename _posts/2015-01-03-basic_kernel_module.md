---
layout: post
title: 基本的なカーネルモジュールの実装
date: 2015-01-03 19:52:38.000000000 +09:00
type: post
published: true
status: publish
categories:
- dev
tags: []
meta:
  _edit_last: '1'
  s4_url2s: ''
  s4_image2s: ''
  s4_ctitle: ''
  s4_cdes: ''
  _layout: inherit
  _syntaxhighlighter_encoded: '1'
  _wp_old_slug: "%e5%9f%ba%e6%9c%ac%e7%9a%84%e3%81%aa%e3%82%ab%e3%83%bc%e3%83%8d%e3%83%ab%e3%83%a2%e3%82%b8%e3%83%a5%e3%83%bc%e3%83%ab%e3%81%ae%e5%ae%9f%e8%a3%85"
author:
  login: admin
  email: hiroki@hirokikana.com
  display_name: hiroki.kana
  first_name: Hiroki
  last_name: Takayasu
---
# はじめに

Open vSwitchのソースを読み進めようと思いたったのですが、カーネルモジュールとして実装されている部分があったのと、その部分が非常に重要になってくる(高速に処理を行うためにカーネルモジュールにしているという売りがあります)ので、今回カーネルモジュールの基本をおさえておこうと思ってまとめた次第です。

カーネルモジュールとは、一般的なプログラムと違い下記のような違いがあります。

  * ユーザー空間ではなくカーネル空間で動作するため、カーネルがアクセスするリソースを使って動作する
  * カーネルに静的リンクする場合と異なり、Linux起動中に動的に機能を追加・削除することができる



カーネルモジュールが必要となるケースの多くは新しいデバイスに対してのデバイスドライバとして利用されることが多く、デバイスの開発者以外の多くの人はユーザー空間で動作するプログラムで十分かもしれません。しかしながら、今回のように読むシチュエーションが出てきた際に非常に役立つと思います。今回は基本的なカーネルモジュールの実装から簡単なキャラクタデバイスを作成するところまでを説明します。

# 最も簡単なカーネルモジュール(Hello World)

まず、カーネルモジュールの開発にあたり必要なカーネルのヘッダやソースなどを導入します。今回はCentOS6.6で行いました。

```bash
$ sudo yum install gcc make kernel-devel kernel-headers  
```

次に任意の場所にカーネルモジュールを開発するためのディレクトリを作り、そこにソースなどのファイルを作成しています。まずはカーネルモジュール本体を実装します。下記が一番簡単なカーネルモジュールのhello.cです。

**hello.c**

```cpp
#include <linux/module.h>
#include <linux/kernel.h>

MODULE_DESCRIPTION("hello kernel module");
MODULE_AUTHOR("hiroki.kana@gmail.com");
MODULE_LICENSE("GPL");

static int init(void)
{
    printk("hello module is loaded\n");
    return 0;
}

static void cleanup(void)
{
    printk("hello module is unloaded\n");
}

module_init(init);
module_exit(cleanup);
```
  
これはモジュールを登録および削除時に文字列をログに出力するだけのカーネルモジュールです。

1行目と2行目は必要なヘッダをincludeしています。3行目〜5行目はモジュールの概要、開発者、ライセンスに関して記述しています。これらの情報はコンパイル後のkoファイルからmodinfoコマンドで参照することができます。

```cpp
$ modinfo hello.ko
filename:       hello.ko
license:        GPL
author:         hiroki.kana@gmail.com
description:    hello kernel module
srcversion:     D2E166AEEDC9CC2AD8EE760
depends:       
vermagic:       2.6.32-504.3.3.el6.x86_64 SMP mod_unload modversions
```

init関数およびcleanup関数は後ほどmodule_init / module_exitで登録をするモジュールを追加・削除した際に行う処理です。登録の際には正常な処理が行えた場合は0を返します。それぞれの関数内で呼ばれているprintkはカーネルコンソールに出力するための関数です。これらはdmesgで参照することができます。カーネルモジュールの実装はこれで以上です。次にコンパイルするためにMakefileを作成します。

**Makefile**
```
obj-m := hello.o
clean-files := *.o *.ko *.mod.[co] *~
all:
    make -C /lib/modules/$(shell uname -r)/build M=$(PWD) modules
clean:
    make -C /lib/modules/$(shell uname -r)/build M=$(PWD) clean
```
---  
  
ディレクトリにはhello.cとMakefileの2つが出来上がったと思います。これで準備は完了です。次にコンパイルを行います。ディレクトリ内で引数無しでmakeコマンドを実行すると下記のような出力がされ、hello.koファイルが作成されると思います。

```bash
$ make
make -C /lib/modules/2.6.32-504.3.3.el6.x86_64/build M=/home/cloud-user/hello_module modules
make[1]: ディレクトリ /usr/src/kernels/2.6.32-504.3.3.el6.x86_64' に入ります
  CC [M]  /home/cloud-user/hello_module/hello.o
  Building modules, stage 2.
  MODPOST 1 modules
  CC      /home/cloud-user/hello_module/hello.mod.o
  LD [M]  /home/cloud-user/hello_module/hello.ko.unsigned
  NO SIGN [M] /home/cloud-user/hello_module/hello.ko
make[1]: ディレクトリ /usr/src/kernels/2.6.32-504.3.3.el6.x86_64' から出ます
```
  
エラーがでなければカーネルモジュールのコンパイルは終了です。最後にカーネルモジュールの追加と削除を行い、dmesgでログが正しく出ているかを確認します。

```bash
$ sudo insmod hello.ko
$ lsmod |grep hello
hello                    985  0
$ dmesg |grep hello
hello module is loaded
$ sudo rmmod hello
$ dmesg |grep hello
hello module is loaded
hello module is unloaded  
```  
  
カーネルモジュールの追加削除にはroot権限が必要でinsmodで追加rmmodで削除を行います。カーネルモジュールの追加を行って正しく組み込まれたかはlsmodコマンドを利用して確認することができます。

# ランダムなアルファベットを返すキャラクタデバイスの作成

カーネルモジュールを作成しましたが、次は簡単なキャラクタデバイスのドライバを作成してみます。Linuxにはキャラクタデバイスとブロックデバイスと呼ばれるものがあり、キャラクタデバイスは1文字ずつ文字を送るようなデバイスで、テキストコンソールやシリアルポートなどもキャラクタデバイスの一種です。一方、ブロックデバイスはディスクなどのブロックと呼ばれる単位で読み込みや書き込みを行うデバイスです。今回は実装が簡単なキャラクタデバイスのドライバを作成しました。

実際のコードはgithub上(<https://github.com/hirokikana/randchar>)においてあります。

今回新たに実装したポイントはopen / close / readのシステムコールでデバイスが呼び出された場合の動作をrandchar_open / randchar_release / randchar_readで実装しています。それぞれのシステムコールで呼び出された場合にどの関数を呼び出すかどうかはfile_operations構造体に登録します。

**randchar.c (open / close / readの実装)**

```cpp
static int randchar_open( struct inode* inode, struct file* filep )
{
    printk( KERN_INFO "%s:open() called\n", module_name );
    
    spin_lock(&randchar_spin_lock);
    if ( access_num ) {
        spin_unlock (&randchar_spin_lock);
        return -EBUSY;
    }
    
    access_num++;
    spin_unlock(&randchar_spin_lock);
    
    return 0;
}

static int randchar_release( struct inode* inode, struct file* filep )
{
    printk( KERN_INFO "%s:close() called\n", module_name );
    spin_lock(&randchar_spin_lock);
    access_num--;
    spin_unlock(&randchar_spin_lock);
    return 0;
}

static ssize_t randchar_read( struct file* filep, char* buf, size_t count, loff_t* pos)
{
    unsigned int i;
    char* alpha = "ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz";
    get_random_bytes(&i, 1);
    if (copy_to_user(buf, &alpha[i % strlen(alpha)], 1) ) {
        printk(KERN_INFO "%s:copy_to_user failed\n", module_name);
        return -EFAULT;
    }
    return 1;
}

static struct file_operations randchar_fops =
{
    owner :THIS_MODULE,
    read :randchar_read,
    open :randchar_open,
    release :randchar_release,
};  
```  
  
今回作るrandcharデバイスは複数のプロセスで開けないようにしました。そのため、アクセスしているデバイス数をaccess_numに保存します(実際は1もしくは0になります)。access_numの内容を変更する際に競合しないようにspin_lockを利用しています。randchar_readではアルファベットの中からランダムな1文字を返すようにするために、ユーザーバッファにランダムな文字列をコピーし、returnでコピーしたサイズ(今回の場合は必ず1文字)を返します。countにはreadシステムコールで呼ばれた際に返すべきバイト数が入っていますが今回は無視しています。

今回、利用するデバイスファイルは/dev/randcharという名前でメジャー番号(デバイスを認識する番号)を77としました。下記コマンドをrootユーザーで作成することで利用できるようになります。

```bash
$ sudo mknod /dev/randchar c 77 0
```
  
デバイス名とメジャー番号はrandchar_initのregister_chrdevでキャラクタデバイスを登録し、モジュール削除時にunregister_chrdevでキャラクタデバイスの登録を解除します。

**randchar.c (キャラクタデバイスの登録・削除)**
```cpp
static int randchar_init( void )
{
    if ( register_chrdev(devmajor, devname, &randchar_fops )) {
        printk( KERN_INFO "%s:register randchar failed\n", module_name);
        return -EBUSY;
    }
    spin_lock_init(&randchar_spin_lock);
    printk(KERN_INFO "%s: loaded into kernel\n", module_name);
    return 0;
}

static void randchar_cleanup(void)
{
    unregister_chrdev(devmajor, devname);
    printk( KERN_INFO "%s: removed fron kernel\n", module_name);
}  
```  
  
実際に利用する際にはhello.cと同じくmakeをした後にinsmodし、作成した/dev/randcharに対して読み込みを行うことでランダムなアルファベットを返します。

```bash
$ make
$ sudo mknod /dev/randchar c 77 0
$ sudo insmod randchar.ko
$ cat /dev/randchar | fold -c10 |head -n1
jpdjxnwTmo
```
  
上記ではfoldで10文字区切りにし、headで1行目を取得するようにしています。これでランダムな10文字のアルファベットが取得できます。

# おわりに

今回はカーネルモジュールの基本的な実装方法についてまとめました。冒頭で書いたとおり、デバイスドライバなどを実装するシチュエーションは少なく、いくら読む必要があるからカーネルモジュールやデバイスドライバの開発を行うためのモチベーションを維持するのは難しいと思います。この記事を書くのにあたり参考にした「RaspberryPiで学ぶ ARMデバイスドライバープログラミング 」という書籍ではRaspberry Piにつないだ7セグメントLEDなどの操作を行うためのデバイスドライバを書く方法が説明されており、実際に開発したデバイスドライバでハードウェアが動作するところまで体験することができ、カーネルモジュールやデバイスドライバの開発のモチベーションを維持し続けることができると思います。特に7セグメントLEDのダイナミック駆動時にユーザー空間で動作させる場合にはチラつきが出てしまうため、カーネルモジュールで実装する必然性というのが出てきており、カーネルモジュールで実装する必要性を感じることができ、学習するうえで非常に良いと感じました。とはいえ私もまだ読み切っていないので、すべてを読み切ってからまたまとめようと思います。

<iframe style="width:120px;height:240px;" marginwidth="0" marginheight="0" scrolling="no" frameborder="0" src="//rcm-fe.amazon-adsystem.com/e/cm?lt1=_blank&bc1=000000&IS2=1&bg1=FFFFFF&fc1=000000&lc1=0000FF&t=hirokikana-22&o=9&p=8&l=as4&m=amazon&f=ifr&ref=as_ss_li_til&asins=488337940X&linkId=5373079cca4d637aef235a65b8ceb665"></iframe>

この記事ではカーネルモジュールの基本的な部分しか説明することができていませんが「カーネルモジュールっていうだけでなんとなく難しい」や「デバイス開発者以外関係ないでしょ」と思っていた方が意外と基本的な部分は単純であると感じてもらえたら幸いです。そんな私もまだ実装されたカーネルモジュールを読んだわけではないので、これをきっかけにOpen vSwitchのソースコードを読み進めていこうと思います。



