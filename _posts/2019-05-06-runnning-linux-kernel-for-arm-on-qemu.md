---
layout: post
title: LinuxカーネルをARM向けにビルドしてQEMUで起動する
date: 2019/05/06 15:45:00 +09:00
type: post
published: true
status: publish
categories:
- dev
tags:
- Linux
---
## はじめに
Linuxのカーネルは誰でも参照できる形でソースコードが公開されています。そのため自らビルドを行い適用することができます。
しかし、私はほとんどの場合ディストリビューションで提供しているカーネルのアップデートパッチを当てるという以上のことをするというのは少なく、Linuxカーネルをビルドして起動するということを行う機会がほとんどありませんでした。

今回はLinuxカーネルのビルド・動作確認方法に関する理解とLinux起動プロセスの全体像を把握するため、可能な限り最低限の手順で最新版のカーネルを起動プロセスを理解するために行った内容を紹介します。せっかくなのでx86ではなくARM向けにビルドするためカーネルをクロスコンパイルを行う手順も合わせて行うこととしました。また、実際にハードウェアで確認するのは面倒なのでQEMUで確認します。

## 準備
なんらかのPCと仮想マシンを起動できる環境があればこの手順は実行することができます。
私の場合は下記のような環境で行いました。

 * Hardware
   * Mac mini(2018)
     * CPU: Core i7 3.2GHz
     * Memory: 32GB
     * OS: macOS 10.14.3 Mojave
 * Software
   * VirtualBox 5.2.22
   * Vagrant 2.2.1

仮想マシンを1台用意するために下記のようなVagrantfileを利用しました。仮想マシン上のディストリビューションはUbuntu 16.04です。
```:Vagrantfile
Vagrant.configure("2") do |config|
  config.vm.box = "bento/ubuntu-16.04"
  config.vm.provider "virtualbox" do |vb|
    vb.memory = "8192"
  end
  config.vm.provision "shell", inline: <<-SHELL
    apt-get update
    apt-get upgrade
  SHELL
end
```

仮想マシンを`vagrant up`で起動し`vagrant ssh`で仮想マシンに入れることを確認します。
仮想マシンでは下記コマンドでQEMUとカーネルのビルド/クロスコンパイルに必要なものをインストールしておきます。
```
sudo apt-get install -y qemu gcc-arm-linux-gnueabi build-essential bison flex libncurses-dev libelf-dev
```
 
## Linuxカーネルのビルド

### エミュレーション可能なボードとDevice Tree
ARMといっても実際にARMが実装されているボードによってハードウェア固有の情報が異なります。そのために作業にあたりQEMUでエミュレート可能なボードを選択するのとそのボードのDevice Treeを用意しておく必要があります。

まず、QEMUでは下記コマンドで対応しているボードを確認することができます。
```
$ qemu-system-arm -M ?
Supported machines are:
akita                Sharp SL-C1000 (Akita) PDA (PXA270)
borzoi               Sharp SL-C3100 (Borzoi) PDA (PXA270)
canon-a1100          Canon PowerShot A1100 IS
cheetah              Palm Tungsten|E aka. Cheetah PDA (OMAP310)
collie               Sharp SL-5500 (Collie) PDA (SA-1110)
connex               Gumstix Connex (PXA255)
cubieboard           cubietech cubieboard
highbank             Calxeda Highbank (ECX-1000)
imx25-pdk            ARM i.MX25 PDK board (ARM926)
integratorcp         ARM Integrator/CP (ARM926EJ-S)
kzm                  ARM KZM Emulation Baseboard (ARM1136)
lm3s6965evb          Stellaris LM3S6965EVB
lm3s811evb           Stellaris LM3S811EVB
mainstone            Mainstone II (PXA27x)
midway               Calxeda Midway (ECX-2000)
musicpal             Marvell 88w8618 / MusicPal (ARM926EJ-S)
n800                 Nokia N800 tablet aka. RX-34 (OMAP2420)
n810                 Nokia N810 tablet aka. RX-44 (OMAP2420)
netduino2            Netduino 2 Machine
none                 empty machine
nuri                 Samsung NURI board (Exynos4210)
realview-eb          ARM RealView Emulation Baseboard (ARM926EJ-S)
realview-eb-mpcore   ARM RealView Emulation Baseboard (ARM11MPCore)
realview-pb-a8       ARM RealView Platform Baseboard for Cortex-A8
realview-pbx-a9      ARM RealView Platform Baseboard Explore for Cortex-A9
smdkc210             Samsung SMDKC210 board (Exynos4210)
spitz                Sharp SL-C3000 (Spitz) PDA (PXA270)
sx1                  Siemens SX1 (OMAP310) V2
sx1-v1               Siemens SX1 (OMAP310) V1
terrier              Sharp SL-C3200 (Terrier) PDA (PXA270)
tosa                 Sharp SL-6000 (Tosa) PDA (PXA255)
verdex               Gumstix Verdex (PXA270)
versatileab          ARM Versatile/AB (ARM926EJ-S)
versatilepb          ARM Versatile/PB (ARM926EJ-S)
vexpress-a15         ARM Versatile Express for Cortex-A15
vexpress-a9          ARM Versatile Express for Cortex-A9
virt                 ARM Virtual Machine
xilinx-zynq-a9       Xilinx Zynq Platform Baseboard for Cortex-A9
z2                   Zipit Z2 (PXA27x)
```
今回は `versatilepb` が参考サイトでもよく使われていたのでこれを利用します。

Device Treeはボード固有のハードウェア情報を記述したもので、これにより様々なボードで同じLinuxカーネルを利用することができるようになります。Device Treeが利用される以前はLinuxカーネルに直接ボード固有のハードウェア情報が記述されていてメンテナンスを継続するのが困難だったという状況だったようです。今回はQEMUで起動する際にカーネルに同梱されているDevice Treeが記述されたバイナリ(dtb: Device Tree Blob)を渡して起動を行います。

### ビルド
まずはLinuxカーネルをcloneし、`versatiblepb` 向けのデフォルト設定を適用します。ちなみにここで指定できるデフォルト設定は `arch/arm/configs/` から参照することができます。

```bash
git clone https://github.com/torvalds/linux.git kernel  
cd kernel
make ARCH=arm versatile_defconfig
```

次に`make ARCH=arm menuconfig` を利用してカーネルパラメータを編集します。
 * `Device Drivers -> Graphics support -> Direct Rendering Manager (XFree86 4.1.0 and higher DRI support)` のチェックを外す
   * 起動時に初期化処理でTimeout待ちをしているようだったのと、特に今回は利用する予定がないので無効

編集が終わったら `Save` で `.config` に保存したうえで `Exit` します。


そしてカーネルをビルドします。マルチコア環境では `-j` オプションを指定して並列で実行する方が良いと思います。
```
make ARCH=arm CROSS_COMPILE=arm-linux-gnueabi- zImage dtbs
```

ビルドが終わったらzImageとdtbがそれぞれ生成されていることを確認します。
```
ls arch/arm/boot/dts/versatile-pb.dtb
ls arch/arm/boot/zImage 
```

## 動作確認

### Linuxの起動プロセスとQEMUでの例
動作確認の前にBIOSを利用したLinuxの起動プロセスについて整理したいと思います。PC上のLinuxは下記のようなプロセスで起動が行われます。
 1. BIOSがMBRにあるブートローダをメモリ上に読み込む
 1. ブートローダーがディスクのファイルシステム等をMBRのパーティションマップから解釈する
 1. ブードローダーがinitrd（初期RAMディスク）等のオプションを指定してカーネルを起動
 1. 初期RAMディスクをrootファイルシステムとしてマウントし初期化プロセス(/sbin/init)を実行
 1. 初期化プロセスが必要な処理を実行しrootファイルシステムをディスクにあるものに変更

この際にブートローダー、カーネル、初期化プロセスはそれぞれ依存関係はありません。このことから下記のような内容で動作確認を行っていきます。
 * カーネルを起動するが初期化プロセスが無い状態
 * 初期化プロセスが何もしないプログラムである状態
 * BusyBoxを利用して初期化プロセスを起動しシェルを利用できる状態
 
また、QEMUにおいてはブートローダを用意する必要はなく `-kernel` / `-initrd` オプションを指定することでカーネルや初期RAMディスクの指定をすることができます。


### カーネルを起動する
まずは初期化プロセスが無い状態でLinuxカーネルを起動していきたいと思います。下記のコマンドでビルドしたカーネルで起動することができます。
```
qemu-system-arm -M versatilepb -kernel ./arch/arm/boot/zImage -dtb ./arch/arm/boot/dts/versatile-pb.dtb -nographic -append "console=ttyAMA0"
```
主なオプションの内容は下記の通りです。
 * -M : 利用するボードを指定
 * -kernel : 起動するカーネルを指定
 * -dtb : 利用するDevice Tree Blobファイルを指定

起動するとrootファイルシステムが無いというメッセージとともにKernel panicが発生したのを確認できると思います。`C-a x` を押すとQEMUを終了することができる
```
Exception stack(0xc7821fb0 to 0xc7821ff8)
1fa0:                                     00000000 00000000 00000000 00000000
1fc0: 00000000 00000000 00000000 00000000 00000000 00000000 00000000 00000000
1fe0: 00000000 00000000 00000000 00000000 00000013 00000000
---[ end Kernel panic - not syncing: VFS: Unable to mount root fs on unknown-block(0,0) ]---
```

### 初期化プロセスを作成する
次に`Hello World` とメッセージを出すだけの初期化プロセス用のプログラムを用意し起動します。起動プロセスが終了するとKernel panicになってしまうため無限ループにしておきます。

```test.c
#include <stdio.h>

int main(int argc, char* argv[])
{
	printf("Hello world\n");
        while(1);
}
```

ARM向けのバイナリとしてコンパイルし、QEMUを使ってコンパイルしたバイナリが正しく動作することを確認します
```
arm-linux-gnueabi-gcc -static -g test.c -o test
qemu-arm -L /usr/arm-linux-gnueabi test
```

次に初期RAMディスクとして利用できるようにcpioを利用してイメージを作成します。
```
echo test | cpio -o --format=newc > rootfs
```

用意したイメージを `-initrd` に指定し起動します。`Hello World`と出力されKernel panicを起こさないかったら正しく動作しています。
```
qemu-system-arm -M versatilepb -kernel ./arch/arm/boot/zImage -dtb ./arch/arm/boot/dts/versatile-pb.dtb -nographic -append "console=ttyAMA0 root=/dev/ram0 rdinit=/test" -initrd rootfs
```

### シェルを利用できるようにする
本来はここから先で必要なコマンド類をインストールしていくということになるわけですが、それはとても手間がかかります。
そのため今回は様々なUNIXコマンドを1つのバイナリの形で提供しているBusyBoxを利用します。サイズも小さく、様々なコマンドを用意する手間も無いので非常に手軽に用意することができます。
まずはcloneをしソースを取得します。
```
git clone git://git.busybox.net/busybox
cd busybox
make ARCH=arm CROSS_COMPILE=arm-linux-gnueabi- defconfig
make ARCH=arm CROSS_COMPILE=arm-linux-gnueabi- menuconfig
```
設定で一部のパラメータを編集します。
 * `Settings -> Build static binary (no shared libs)` をチェックする
   * 単一のバイナリで動作するように静的リンクを行う
 * `Networking Utilities -> Enable IPv6 support` のチェックを外す
   * ビルドするときにエラーになってしまって、今回はIPv6は利用しないのでビルドの対象から外した

最後にビルドを行い、installを行うと`_install` 以下にrootファイルシステムとして利用できる形でファイルが生成されます。
```
make ARCH=arm CROSS_COMPILE=arm-linux-gnueabi- 
make ARCH=arm CROSS_COMPILE=arm-linux-gnueabi- install
```

その他必要なデバイスファイルを含んだディレクトリやファイルを生成します。
```
cd _install/

mkdir dev
sudo mknod dev/null c 1 3

mkdir proc
mkdir sys
mkdir -p etc/init.d
```
/etc/init.d/rcS
```/etc/init.d/rcS
#!/bin/sh
mount -t proc none /proc
mount -t sysfs none /sys
/sbin/mdev -s
```
/etc/inittab
```/etc/inittab
::sysinit:/etc/init.d/rcS 
::respawn:-/bin/sh 
::respawn:/sbin/getty -L ttyS5 115200 vt100 
::ctrlaltdel:/bin/umount -a -r
```

最後にrootファイルシステムとして利用するためにイメージを作るために `_install` ディレクトリ以下で下記を実行します。
```
find . | cpio -o --format=newc > ../rootfs
```

最後にQEMUを起動しシェルが利用できるか確認します。
```
qemu-system-arm -M versatilepb -kernel /path/to/arch/arm/boot/zImage -dtb /path/to/arch/arm/boot/dts/versatile-pb.dtb -nographic -append "console=ttyAMA0 root=/dev/ram0 rdinit=/sbin/init" -initrd rootfs
```

下記のようにビルドしたバージョンのkernelで実行されていることがわかります。
```
/ # uname -a
Linux (none) 5.1.0-rc7+ #7 Mon May 6 03:42:54 UTC 2019 armv5tejl GNU/Linux
```

## おわりに
実際のハードウェアで起動するには至りませんでしたが、Linuxカーネルのビルドと正しくビルドできたかということに関して確認できるような手順を整理することができました。
この次の流れとしては[Linux From Scratch](http://www.linuxfromscratch.org/lfs/)の手順でBusyBoxを使わずに環境を準備したりするのが良いのではないかという気がしています。


## 参考リンク
 - [Cross-compile Linux kernel for ARM and run on QEMU](https://designprincipia.com/compile-linux-kernel-for-arm-and-run-on-qemu/)
 - [QEMU user-mode emulation for ARM - Qiita](https://qiita.com/tobira-code/items/f76610983902a4ae2441)
 - [Linux を(わりと)シンプルな構成でビルドして Qemu で起動する - Qiita](https://qiita.com/kaishuu0123/items/33621af2aca44d8d2c72)
 - [Compiling Linux kernel for QEMU ARM emulator \| Freedom Embedded](https://balau82.wordpress.com/2010/03/22/compiling-linux-kernel-for-qemu-arm-emulator/)
 - [Atelier Orchard: ARM QEMUでlinuxカーネルを起動](http://atelier-orchard.blogspot.com/2013/02/arm-qemulinux.html)
 - [BusyBoxのコンパイル - keropoの備忘録](http://keropo.hatenadiary.com/entry/2013/04/17/230405)
 - [Linuxがブートするまで &middot; Keichi Takahashi](https://keichi.net/post/linux-boot/)
