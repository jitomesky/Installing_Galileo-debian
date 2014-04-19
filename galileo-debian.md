# How to Install Debian(wheezy) on Galileo

## 概要
Intel Galileoに、Arduino機能を保ったままDebian（wheezy）をインストールする方法の解説です。

この方法でインストールすれば、Debianインストール後もArduino IDEを使ってプログラムを書き込み、実行させることができます。

ただしいくつか既知の問題や、検証不足な部分があるので、問題解決に協力していただけると嬉しいです。


## Debianのインストール
GalileoにDebianをインストールする方法は、主に以下のページにまとまっています。

[Installing ROS on Galileo: The Debian way](http://wiki.ros.org/IntelGalileo/Debian)

基本的にはこのページを参考に進めていきます。


### Yoctoイメージの取得
今回使用するDebianは、Intelが配布しているカーネルや、その他モジュールを使用します。
以下のコマンドを実行して、それらをダウンロードしてください。

```bash
$ mkdir ~/Downloads
$ cd ~/Downloads
$ wget http://downloadmirror.intel.com/23171/eng/LINUX_IMAGE_FOR_SD_Intel_Galileo_v0.7.5.7z
$ sudo apt-get -y install p7zip-full
$ 7z x LINUX_IMAGE_FOR_SD_Intel_Galileo_v0.7.5.7z
```


### モジュールを移植する
配布されているLinuxイメージから、必要なモジュールを取り出して、Debianイメージのなかに追加します。
以下のコマンドを実行してください。

```
$ cd
$ mkdir galileo-debian
$ cd gaileo-debian
$ mkdir mnt-loop sdcard image
$ dd if=/dev/zero of=loopback.img bs=1G count=3
$ mkfs.ext3 loopback.img
```
ここで次のように聞かれるので、yを押してください。
```
mke2fs 1.42.5 (29-Jul-2012)  
loopback.img is not a block 
Proceed anyway? (y,n) 
```
その後は以下のコマンドを実行していってください。

```
$ sudo mount -o loop loopback.img mnt-loop
$ sudo aptitude -y install debootstrap
$ sudo debootstrap --arch i386 wheezy ./mnt-loop http://http.debian.net/debian/ 
$ sudo mount -o loop ~/Downloads/LINUX_IMAGE_FOR_SD_Intel_Galileo_v0.7.5/image-full-clanton.ext3 image
$ sudo cp -r image/lib/modules mnt-loop/lib
$ cd mnt-loop
$ sudo mkdir media sketch
$ sudo mkdir opt/cln
$ cd media
$ sudo mkdir card cf hdd mmc1 net ram realroot union
$ cd ../dev
$ sudo mkdir mtdblock0 mtd0 
$ cd ../..
```

### initスクリプトの追加と修正
いくつか移動しきれていないスクリプトがあったので、移動します。
以下のコマンドを実行してください。

```
$ sudo cp image/etc/init.d/clloader.sh mnt-loop/etc/init.d
$ sudo cp image/etc/init.d/start-clloader.sh mnt-loop/etc/init.d/
$ sudo cp image/etc/init.d/sysfs* mnt-loop/etc/init.d/
$ cd mnt-loop/etc/rcS.d
$ sudo ln -s ../init.d/sysfs.sh S02sysfs.sh
$ sudo ln -s ../init.d/sysfs_devtmpfs.sh S02sysfs_devtmpfs.sh
$ cd
```

次に ``` /opt/cln/galileo ``` 以下をコピーし、inittabを編集してランレベルの設定などを行います。

```
$ sudo cp -avr image/opt/cln mnt-loop/opt
$ sudo sh -c "echo 'grst:5:respawn:/opt/cln/galileo/galileo_sketch_reset -v' >> mnt-loop/etc/inittab"
$ sudo sh -c "echo 'clld:5:respawn:/etc/init.d/clloader.sh' >> mnt-loop/etc/inittab"
$ sudo sh -c "cat image/etc/modules-load.d/auto.conf >> mnt-loop/etc/modules"
$ sudo sed -i -e "s/id:2:initdefault:/id:5:initdefault:/" mnt-loop/etc/inittab
```

### カーネルモジュールの追加
カーネルモジュールを読み込む設定を引き継ぎます
以下のコマンドを実行してください。

```
$ sudo sh -c "cat image/etc/modules-load.d/auto.conf >> mnt-loop/etc/modules"
```

### ttyGS0の復活
USB-CLIENTから出ているシリアルポートが登録されていないので、再登録します。
以下のコマンドを実行してください。

```
$ sudo sh -c "echo 'KERNEL=="ttyGS*",  NAME="%k", GROUP="uucp", MODE="0666"' > mnt-loop/etc/udev/rules.d/50-udev.rules"
```


### 初期設定を行う
ここからはchrootで実行環境をイメージ上に移し、ユーザーの追加などの細かい作業を行います。
まずは以下のコマンドを実行してください。

```
$ sudo umount image
$ sudo mount -t proc proc mnt-loop/proc
$ sudo mount -t sysfs sysfs mnt-loop/sys
$ sudo chroot mnt-loop /bin/bash
```

これでGalileo用Debianイメージがマウントされ、そこに実行環境が移されました。

### usleepコマンドの置き換え
clloader.shのusleepコマンドはdebianにはないので、コンパチなパッケージを入れて置き換えます。
以下のコマンドを実行してください。

```
# aptitude -y install sleepenh
# sed -i -e "s/usleep 200000/sleepenh 0.2/" /etc/init.d/clloader.sh
```

### killallコマンドの追加
現状killallが使えないので、追加します。

```
# aptitude -y install psmisc
```


#### rootパスワードの設定
rootのパスワードを設定します。
以下のコマンドを実行し、パスワードを入力して、Enterを押してください
（画面上に「●」などは表示されませんが、きちんと入力できています）。

```
# passwd
```

#### ホストネームの設定
ホストネームを設定します。
ホストネームは ```/etc/hostname``` に記されており、デフォルトでは「debian」となっているので、これをclantonに変更します。

```
# echo "clanton" > /etc/hostname
```

#### コンソールの設定
コンソールを有効にするため、 ```/etc/inittab``` を開いて以下の行のコメントを外してください。

```
#T1:23:respawn:/sbin/getty -L ttyS1 9600 vt100
```

そして以下のようにランレベルを2-5、ボーレートを9600から115200に変更してください。

```
T1:2345:respawn:/sbin/getty -L ttyS1 115200 vt100
```

以下のコマンドを実行しても、同じ結果が得られます。

```
# sed -i -e "s/#T1:23:respawn:\/sbin\/getty -L ttyS1 9600 vt100/T1:2345:respawn:\/sbin\/getty -L ttyS1 115200 vt100/" /etc/inittab
```

#### SSHのダウングレード
標準のSSHは動かないようなので、1つ前のバージョンに戻します。
以下のコマンドを実行してください。

```
# apt-get purge openssh*
# groupadd -g 35 sshd
# useradd -u 35 -g 35 -c sshd -d / sshd
# apt-get update
# apt-get -y install screen build-essential zlib1g-dev libssl-dev 
# cd /root
# wget http://www.mirrorservice.org/pub/OpenBSD/OpenSSH/portable/openssh-5.9p1.tar.gz
# tar zxvf openssh-5.9p1.tar.gz
# cd openssh-5.9p1
# ./configure
# make
# make install
```

次に自動起動のためのスクリプトを作ります。
以下の内容を ``` /etc/init.d/ssh ``` として保存してください。

```
#!/bin/sh
case "$1" in
  start)
    echo "Starting script ssh "
    /usr/local/sbin/sshd
    ;;
  stop)
    echo "Stopping script ssh"
    kill -9 `pgrep sshd`
    ;;
  *)
    echo "Usage: /etc/init.d/ssh {start|stop}"
    exit 1
    ;;
esac

exit 0
```

ファイルができたら、これの自動起動を有効化します。
以下のコマンドを実行してください。

```
# chmod 755 /etc/init.d/ssh
# update-rc.d ssh defaults
```

#### Gitのダウングレード
SSHと同様にGitもダウングレードしたほうが良いとのことなので、ダウングレードします。
以下のコマンドを実行してください。

```
# cd /root
# apt-get -y install liblocale-msgfmt-perl gettext libcurl4-openssl-dev curl ntpdate unzip libexpat1-dev python
# wget https://github.com/git/git/archive/v1.7.0.9.zip
# unzip v*
# cd git-*
# make prefix=/usr/local all 
# make prefix=/usr/local install
```

### udevの禁止
``` core-image-minimal-initramfs-clanton.cpio.gz ``` 内にあるudevと、Debianのudevが2重起動してしまって、IOエキスパンダが認識しなくなるので、debian側のudevを殺します。
以下のコマンドを実行してください。

```
# rm -rf /etc/rcS.d/S02udev
```

### clloaderの再コンパイル
移植した``` /opt/cln/galileo``` 以下のバイナリは起動することができません。
なのでソースから入れ直します。
必要なファイルが結構大きいので、一度ホスト環境に戻って作業を行います。

```
# exit
$ wget http://downloadmirror.intel.com/23171/eng/Board_Support_Package_Sources_for_Intel_Quark_v0.7.5_full_yocto_archive.tar.gz

$ tar zxvf Board*
$ sudo cp GPL_COMPLIANCE/source/galileo-target-0.1-r0/galileo-target-0.1-r0-prepatch.tar.gz mnt-loop/root/
$ sudo chroot mnt-loop /bin/bash
# cd /root
# tar zxvf galileo-target-0.1-r0-prepatch.tar.gz
# cd galileo*c
# make
# make install
# cp clloader galileo_sketch_reset /opt/cln/galileo/
```

### i586ライブラリの追加
Arduino IDEで作ったプログラムはi586用のライブラリを必要とするので、追加します。
以下のコマンドを実行してください。

```
# exit
$ sudo mkdir mnt-loop/usr/lib/i586-linux-gnu
$ sudo cp -R image/usr/lib/* mnt-loop/usr/lib/i586-linux-gnu/

$ sudo cp image/lib/l* mnt-loop/lib/

$ sudo cp mnt-loop/etc/ld.so.conf.d/i486-linux-gnu.conf mnt-loop/etc/ld.so.conf.d/i586-linux-gnu.conf 
$ sudo sed -i -e "s/4/5/" mnt-loop/etc/ld.so.conf.d/i586-linux-gnu.conf

$ sudo chroot mnt-loop /bin/bash
# ldconfig
```


#### ユーザーの追加とパスワードの設定
一般ユーザーの追加とパスワードの設定を行います。

以下のコマンドを、galileoを自分の好きな名前に置き換えながら実行してください。

```
# apt-get -y install sudo
# useradd galileo -m -s /bin/bash
# passwd galileo
# usermod -aG sudo galileo
# mkdir /home/galileo
# chown galileo /home/galileo
# su galileo
$ bash
```

なお、ここで作ったユーザーはsudoerに追加され、sudoが使用できます。


#### ネットワークの設定
ネットワーク・インターフェースの追加と、ホストの設定を行います。

``` /etc/network/interfaces ``` を以下のように書き換えてください。

```
# interfaces(5) file used by ifup(8) and ifdown(8)
auto lo
iface lo inet loopback
auto eth0
iface eth0 inet dhcp
```

なお、以下のコマンドを実行しても同じ結果が得られます。

```
$ sudo sh -c "echo 'auto eth0' >> /etc/network/interfaces"
$ sudo sh -c "echo 'iface eth0 inet dhcp' >> /etc/network/interfaces"
```

これで有線LAN（eth0）がDHCPで立ち上がるようになります。

次にローカルホストの設定をします。
``` /etc/hosts ``` を次のように書き換えてください。

```
127.0.0.1       localhost clanton
```

なお、次のコマンドを実行しても同じ結果が得られます。

```
$ sudo sed -i -e "s/127.0.0.1\tlocalhost/127.0.0.1\tlocalhost clanton/" /etc/hosts
```


### NTPDの設定
Galileoの時刻を自動修正してくれるように、NTPデーモンを立てます。
以下のコマンドを実行してntpをインストールしてください。

```
$ sudo aptitude -y install ntpd
```

次に設定を修正してNICTから時刻をあわせるようにします。

以下のコマンドで設定ファイルを開いてください。

```
% sudo nano /etc/ntp.conf
```

そしてserverの記述を以下のようにコメントアウトします

```
#server 0.debian.pool.ntp.org iburst
#server 1.debian.pool.ntp.org iburst
#server 2.debian.pool.ntp.org iburst
#server 3.debian.pool.ntp.org iburst
```

そしてその下に以下を追加してください。

```
pool ntp.nict.jp iburst
```

このiburstがないと起動直後の時計が大きく狂うようです。

編集が完了したらCtrl+O,Ctrl+Xで保存して抜けてください。nanoは画面下にコマンドが出てるので親切ですね。


### イメージのアンマウント
これですべての設定は完了です。
以下のコマンドで一番上まで抜けて、イメージをアンマウントしてください。

```
$ exit
$ exit
# exit
$ sudo umount mnt-loop/proc
$ sudo umount mnt-loop/sys
$ sudo umount mnt-loop
```

### SDカードへのインストール
先ほど作ったイメージをSDカードに入れていきます。

まず先ほどのイメージ(loopback.img)を ``` image-full-clanton.ext3 ``` にリネームしてください。

```
$ mv loopback.img image-full-clanton.ext3
```

このイメージをFATでフォーマットしたSDカードにコピーし、[https://communities.intel.com/docs/DOC-22226](https://communities.intel.com/docs/DOC-22226)から取得した ```LINUX_IMAGE_FOR_SD_Intel_Galileo_v0.7.5.7z``` の中にある以下のファイルを一緒に入れてください。

 * core-image-minimal-initramfs-clanton.cpio.gz
 * bzImage
 * boot


これでインストールは完了です。

## Debianの起動
SDカードをGalileoにさして、電源を入れてください。
2分ほどでシステムが立ち上がり、ネットワークが立ち上がればSSHが一緒に動き出すはずです。

ただしIPアドレスはDHCPで取ってくるようになっているので、シリアルケーブルを使ってコンソールにアクセスして確認して下さい。

### 時刻設定
起動後に以下のコマンドを実行して時刻設定してください。

```
$ sudo ntpdate ntp.nict.jp
```



## 既知の問題
現在、以下の様な問題が見つかっています。

### カーネルモジュールの読み込み失敗
カーネルモジュールの読み込みが、いくつか失敗します。

```
Loading kernel module spi-pxa2xx-pci.
ERROR: could not insert 'spi_pxa2xx_pci': Invalid argument

Loading kernel module spi-lpc-sch.
FATAL: Module spi-lpc-sch not found.
```

この影響がどこに出るのか等は未調査です。



### aptitudeでSegmentation fault
aptitudeでパッケージをインストールすると、インストール完了後に何故か
```
Segmentation fault
```
が出ます。

インストール自体は成功しているので、実害はありませんが……うーん。


### aptitudeでldconfigのworningが大量に出る
aptitudeでライブラリを追加、正確にはldconfigを実行すると以下の様なメッセージが出てきます。

```
Setting up libm17n-0 (1.6.3-2) ...
ldconfig: /lib/libtinfo.so.5 is not a symbolic link

ldconfig: /lib/libudev.so.0 is not a symbolic link

ldconfig: /lib/libc.so.0 is not a symbolic link

ldconfig: /lib/libusb-0.1.so.4 is not a symbolic link

ldconfig: /lib/libusb-1.0.so.0 is not a symbolic link

ldconfig: /lib/libcap.so.2 is not a symbolic link

ldconfig: /lib/libncurses.so.5 is not a symbolic link
```

### 時刻設定の保持
元イメージの ```/etc/init.d/save-rtc.sh``` で時刻を保存、 ``` /etc/init.d/bootmisc.sh ``` で起動時に時刻を読み込んでいるようです。

インストールしたdebianイメージにはこのスクリプトを移植していないので、時刻設定の保持はうまくいかないはずですが………私の環境では再起動後も時刻設定が初期化されていませんでした。

この理由がまだわかっていません。


### 検証不足
Arduino IDEでコンパイル、Arduino IDEでインストール、そして実機で動作することを確認したのは、以下の2サンプルです。

* Basics - Blink
* LiquidCrystal - HelloWorld

その他にはPWMの動作を確認しました。

それ以外の機能については動作確認ができていません。




