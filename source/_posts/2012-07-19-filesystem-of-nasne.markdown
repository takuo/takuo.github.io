---
layout: post
title: nasne のファイルシステムを見てみる
date: 2012-07-19 22:52
comments: true
categories: [ PS3, nasne, trone, Linux ]
---
物理的な分解記事とかよりも面白いと思ったのでちょっとだけnasneの中身をみてました。

<!-- more -->

まずはパーティションの確認から。

    # parted /dev/sdd
    GNU Parted 2.3
    Using /dev/sdd
    Welcome to GNU Parted! Type 'help' to view a list of commands.
    (parted) print
    Model: Hitachi HTS545050A7E380 (scsi)
    Disk /dev/sdd: 500GB
    Sector size (logical/physical): 512B/512B
    Partition Table: msdos
    
    Number  Start   End     Size    Type     File system  Flags
     1      1049kB  269MB   268MB   primary  ext3         boot
     2      269MB   1343MB  1074MB  primary  ext3
     3      1343MB  500GB   499GB   primary  xfs

ほうほう。Linuxっぽい。xfsはきっと録画した実データ置き場ですね。  
mount してみます。

    # mount -o ro /dev/sdd1 /mnt/nasne/1
    # mount -o ro /dev/sdd2 /mnt/nasne/2
    # mount -o ro /dev/sdd3 /mnt/nasne/3

間違えて壊したくないので read-only でマウントしてます。  

    # cd /mnt/nasne
    # find > /tmp/nasne.fs.txt

[nasne.fs.txt](/assets/nasne.fs.txt)  
telnetd とか samba とか興味深いですね。samba は3.0.37でした。  
sbin,binなどにあるプログラムの実体はほぼ全てbusyboxです。  

最初のパーティション(1)にboot関連(多分)のものが入っており、おそらくハードウェア個体に紐付けられてこのnasneでしか起動できないようになってます。  
ファイルの中身は暗号化されており、わかりません。  

この時点ですでに1番組録画してあるのですが、パーティション(3)にある録画データはちゃんと暗号化(きっとハード固有の鍵により)されております。  

アーキテクチャを調べてみます。

    # file 2/lib/libc-2.9.so
    libc-2.9.so: ELF 32-bit LSB shared object, MIPS, MIPS32 rel2 version 1 (SYSV), dynamically linked (uses shared libs), for GNU/Linux 2.6.16, with unknown capability 0xf41 = 0x756e6700, with unknown capability 0x70100 = 0x1040000, not stripped

MIPSだそうです。

    # cat 2/etc/issue
    ********************************************************************************
    *              Welcome to Viper MIPS Linux :-)                                 *
    ********************************************************************************

ほうほう

    # cat 2/etc/passwd
    root::0:0:root:/root:/bin/sh
    www:x:80:80:root:/www:/bin/sh
    nobody:x:99:99:Nobody:/:/sbin/nologin
    bin:x:1:1:bin:/bin:/sbin/nologin
    daemon:x:2:2:daemon:/sbin:/sbin/nologin

ははあ。rootのパスワードはないんですね。

    # cat 2/etc/init.d/rcS
    #!/bin/sh
    trap "" SIGHUP
    #echo "Mounting proc"
    mount -t proc none /proc
    #echo "Mounting usbfs"
    mount -t usbfs none /proc/bus/usb
    #echo "Mounting sysfs"
    mount -t sysfs none /sys
    
    #echo "Starting uevent daemon"
    /usr/bin/uevent_daemon

    #echo "Starting Udev"
    start_udev
    
    #echo "Mounting devpts"
    mount -t devpts none /dev/pts
    
    sysctl -p
    
    #export HOSTNAME="nasne"
    #echo "Setting hostname $HOSTNAME"
    #hostname $HOSTNAME
    
    #echo "Bringing up loopback device"
    ifconfig lo 127.0.0.1 netmask 255.0.0.0 up
    route add -net 127.0.0.0 netmask 255.0.0.0 dev lo
    
    #echo "Bringing up Eth0"
    
    # ---- for DTVTuner ----
    /opt/dtvtuner/etc/startdtvtuner


なるほど。  
ここでinetdとかtelnetdを起動するように変更すれば稼働中のnasneにログインできるかもしれません。  

というわけで、別のHDDに換装するにはまるごとddしてコピーし、partedでxfsのパーティションを適当に大きくしたりすればよさそうです。  
肝心のカーネルイメージが何処にあるのかわかりませんでした。  

**(2012-07-20追記)**  

実際に /usr/sbin/inetd を起動するように書いてみたのですが、どうやらinitramfsかそのへんの処理でどこかに存在するイメージの展開で / を再作成しているようで、編集前の状態に戻ってしまいます。  

試しに dumpe2fs で / の状態を確認すると、

    Filesystem created:       Wed May 16 17:52:14 2012
    Last mount time:          Thu Jan  1 09:00:05 1970
    Last write time:          Thu Jan  1 09:00:05 1970

となっており、mkfsはされてない(初期化ではない)が、システムの起動直後にマウントされ、再作成されてるように思えます。  
そこで、__chattr +i {rootfs}/etc/init.d/rcS__ で書き込み禁止属性をつけ、rcSが書き戻されないようにしてから起動して見ましたが… 起動せず。  
どうやら展開中のエラーで起動処理が止まってるのだと思われます。(__chattr -i__ しなおしたらなおりました)  

そんなわけで、簡単に好きなプログラムを走らせるようにはできておらず、もうひと工夫が必要そうです。(あるいはexploitを見つけるか)  

{% raw %}
<iframe src="http://rcm-jp.amazon.co.jp/e/cm?lt1=_blank&bc1=000000&IS2=1&bg1=FFFFFF&fc1=000000&lc1=0000FF&t=takuojp02-22&o=9&p=8&l=as4&m=amazon&f=ifr&ref=ss_til&asins=B007V9T9ZK" style="width:120px;height:240px;" scrolling="no" marginwidth="0" marginheight="0" frameborder="0"></iframe>
<iframe src="http://rcm-jp.amazon.co.jp/e/cm?lt1=_blank&bc1=000000&IS2=1&bg1=FFFFFF&fc1=000000&lc1=0000FF&t=takuojp02-22&o=9&p=8&l=as4&m=amazon&f=ifr&ref=ss_til&asins=B0034KZXBO" style="width:120px;height:240px;" scrolling="no" marginwidth="0" marginheight="0" frameborder="0"></iframe>
<iframe src="http://rcm-jp.amazon.co.jp/e/cm?lt1=_blank&bc1=000000&IS2=1&bg1=FFFFFF&fc1=000000&lc1=0000FF&t=takuojp02-22&o=9&p=8&l=as4&m=amazon&f=ifr&ref=ss_til&asins=B0035V5EA2" style="width:120px;height:240px;" scrolling="no" marginwidth="0" marginheight="0" frameborder="0"></iframe>
{% endraw %}

