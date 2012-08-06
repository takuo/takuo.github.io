---
layout: post
title: "au IS12Sでスワップを有効にする"
date: 2012-08-07 00:14
comments: true
categories: [ IS12S, Android ]
---
IS12S をICS にアップデートしてからというもの、メモリが足りなくていささか不便になってきました。  
そこでスワップを導入してみることにします。 ※ 要root  

<!-- more -->

そもそも1GBもあるはずなにどうなってるのか調べてみると...

    $ cat /proc/meminfo
    MemTotal:         650404 kB
    MemFree:           20656 kB
    Buffers:             376 kB
    Cached:            71860 kB
    ...

どうやらカーネルに予約されているのか知りませんが650MB弱しか使えないようですね。  
### カーネルの変更は不要、root は必要

幸いなことに、IS12Sのカーネルはswapがを扱えるようになっていますので、カーネルへの変更は不要です。  
root(とbusybox)が必要ですので注意。rootの取得方法は下記URLのサイト様を参考にするとよいです。   

<http://blog.huhka.com/2012/06/rooting-toolkit-for-xperia-acro-hd.html>
<http://blog.huhka.com/2012/07/xperia-acro-hd-kddi-is12sics-root.html>
<http://blog.huhka.com/2012/08/gbis12sicsroot.html>

いずれも2.3.7の状態である必要があるので4.0にしちゃった場合はなんとかして2.3.7に戻す必要があります。(ftfがあれば戻せる)


### どこにスワップ領域を確保するか

さて、スワップを有効にするにはスワップ**パーティション**を作るか、スワップ**ファイル**をつくるかの2通りありますが、SDカードにスワップ**ファイル**を作るのはやめたほうがよさそうです。
(スワップを有効にしたままSDカードをマウント解除すると挙動がおかしくなるため)   
 
スワップ**ファイル**を作る場合は/sdcard/(内蔵ストレージ)とか/data/ のどっかに作ったほうがよいでしょう。
しかし、/sdcard/swapfile のように内蔵ストレージに作ると、後述する手順で起動時に有効にすることができないため、再起動のたびに手動で有効にしなくてはならないことを留意してください。  

また、SDカード上にスワップ**パーティション**を作る場合もクラス10とか速いSDカードを使ったほうがよいでしょう。

### スワップファイルを作る場合

adb shell で作業を行います。 su でrootになる必要があります。  

    $ su
    # dd if=/dev/zero of=/data/local/swapfile bs=$((1024 * 1024)) count=256
    # chmod 644 /data/local/swapfile
    # mkswap /data/local/swapfile
    # swapon /data/local/swapfile

これで 256MBのスワップファイルを作り、有効になりました。  

### SDカードにスワップパーティションを作る場合

少々面倒ですが、adb shell だけでもできます。  
SDカードの中身は全部消えるので注意。  

まず SDカードをアンマウント

    $ su
    # umount /mnt/ext_card

fdisk でパーティションを切ります。 SDカードは /dev/block/mmcblk1 です。

    # fdisk /dev/block/mmcblk1
    
    The number of cylinders for this disk is set to 3801.
    There is nothing wrong with that, but this is larger than 1024,
    and could in certain setups cause problems with:
    1) software that runs at boot time (e.g., old versions of LILO)
    2) booting and partitioning software from other OSs
       (e.g., DOS FDISK, OS/2 FDISK)
    
    Command (m for help): p
    
    Disk /dev/block/mmcblk1: 31.3 GB, 31393316864 bytes
    256 heads, 63 sectors/track, 3801 cylinders
    Units = cylinders of 16128 * 512 = 8257536 bytes
    
                  Device Boot      Start         End      Blocks  Id System
    /dev/block/mmcblk1p1   *           1        3801    30651232+  c Win95 FAT32 (LBA)
    
    Command (m for help): d
    Selected partition 1

    Command (m for help): n
    Command action
       e   extended
       p   primary partition (1-4)
    p
    Partition number (1-4): 1
    First cylinder (1-3801, default 1): Using default value 1
    Last cylinder or +size or +sizeM or +sizeK (1-3801, default 3801): 3768
    
    Command (m for help): n
    Command action
       e   extended
       p   primary partition (1-4)
    p
    Partition number (1-4): 2
    First cylinder (3769-3801, default 3769): Using default value 3769
    Last cylinder or +size or +sizeM or +sizeK (3769-3801, default 3801): Using default value 3801
    
    Command (m for help): p
    
    Disk /dev/block/mmcblk1: 31.3 GB, 31393316864 bytes
    256 heads, 63 sectors/track, 3801 cylinders
    Units = cylinders of 16128 * 512 = 8257536 bytes
    
                  Device Boot      Start         End      Blocks  Id System
    /dev/block/mmcblk1p1               1        3768    30385120+ 83 Linux
    /dev/block/mmcblk1p2            3769        3801      266112  83 Linux
    
    Command (m for help): t 
    Partition number (1-4): 1
    Hex code (type L to list codes): c
    Changed system type of partition 1 to c (Win95 FAT32 (LBA))
    
    Command (m for help): a
    Partition number (1-4): 1
    
    Command (m for help): t
    Partition number (1-4): 2
    Hex code (type L to list codes): 82
    Changed system type of partition 2 to 82 (Linux swap)
    
    Command (m for help): p
    
    Disk /dev/block/mmcblk1: 31.3 GB, 31393316864 bytes
    256 heads, 63 sectors/track, 3801 cylinders
    Units = cylinders of 16128 * 512 = 8257536 bytes
    
                  Device Boot      Start         End      Blocks  Id System
    /dev/block/mmcblk1p1   *           1        3768    30385120+  c Win95 FAT32 (LBA)
    /dev/block/mmcblk1p2            3769        3801      266112  82 Linux swap
    
    Command (m for help): w
    The partition table has been altered!
    
    Calling ioctl() to re-read partition table
    
    # busybox mkdosfs /dev/block/mmcblk1p1
    # mkswap /dev/block/mmcblk1p2 
    Setting up swapspace version 1, size = 272494592 bytes
    # swapon /dev/block/mmcblk1p2

これでだいたい256MBのスワップパーティションが作れました。
一番目のパーティションは通常使用するvfatパーティションです。(boot フラグは不要でしょうが、最初からついてたので付けただけです)
サイズは好きにしたらよいと思いますが、一番目はFAT32にしたほうが無難な気がします。  

あとはスワップの頻度を調整します

    # sysctl -w vm.swappiness=40

値は0から100で100が最も頻度が高くなります。だいたい30くらいがいいようですが、様子を見ながら調整するとよいでしょう。  
カーネルのデフォルトは60です。

### 起動時に有効にするために

IS12S は init.d 的なものがなくユーザー追加のスクリプトを起動時に実行することはできませんが、起動時に **/system/etc/install-recovery.sh** というスクリプトを実行している箇所があるので、これを利用します。  
**/system/etc/install-recovery.sh** そのものは標準では存在しないので新規に作ることになります。
スワップファイルの場合、このスクリプトの実行時には /sdcard がマウントされていないので、/sdcard/ にファイルを作った場合は注意です。

``` sh /system/etc/install-recovery.sh
#!/system/bin/sh

# パーティションの場合
swapon /dev/block/mmcblk1p2
# ファイルの場合
# swapon /data/local/swapfile
sysctl -w vm.swappiness=40

```

こんな感じで **install-recovery.sh** を /sdcard/ に作り、下記のコマンドでコピーします。  

    $ su
    # cd /system/etc
    # mount -o remount,rw /system ; cp /sdcard/install-recovery.sh . ; chmod 755 install-recovery.sh ; mount -o remount,ro /system

一行で実行すればリマウントによる再起動を回避できるかと思われます。

というわけでスワップを有効にできました。

    $ su
    # free
              total         used         free       shared      buffers
      Mem:       650404       622388        28016            0         3128
     Swap:       266108        32388       233720
    Total:       916512       654776       261736
    # cat /proc/meminfo
    MemTotal:         650404 kB
    MemFree:           29596 kB
    Buffers:            3136 kB
    Cached:           175632 kB
    SwapCached:        13048 kB
    Active:           250536 kB
    Inactive:         264296 kB
    Active(anon):     163560 kB
    Inactive(anon):   174236 kB
    Active(file):      86976 kB
    Inactive(file):    90060 kB
    Unevictable:        1228 kB
    Mlocked:               0 kB
    HighTotal:        523264 kB
    HighFree:           6260 kB
    LowTotal:         127140 kB
    LowFree:           23336 kB
    SwapTotal:        266108 kB
    SwapFree:         233728 kB
    ...


効果のほどですが、OOMでバックグラウンド処理が途中で殺されることがなくなった気がします。

{% img https://lh3.googleusercontent.com/-CfWIVT_CFTY/UB_8w-H4_9I/AAAAAAAAKg4/he2aOplKlB4/s288/2012-08-07%252002.18.31.png https://lh3.googleusercontent.com/-CfWIVT_CFTY/UB_8w-H4_9I/AAAAAAAAKg4/he2aOplKlB4/s800/2012-08-07%252002.18.31.png System Tuner %}  
結構使ってますが空きメモリにそこそこ余裕が見られます。

{% raw %}
<iframe src="http://rcm-jp.amazon.co.jp/e/cm?lt1=_blank&bc1=000000&IS2=1&bg1=FFFFFF&fc1=000000&lc1=0000FF&t=takuojp02-22&o=9&p=8&l=as4&m=amazon&f=ifr&ref=ss_til&asins=B0065SI5SK" style="width:120px;height:240px;" scrolling="no" marginwidth="0" marginheight="0" frameborder="0"></iframe>
<iframe src="http://rcm-jp.amazon.co.jp/e/cm?lt1=_blank&bc1=000000&IS2=1&bg1=FFFFFF&fc1=000000&lc1=0000FF&t=takuojp02-22&o=9&p=8&l=as4&m=amazon&f=ifr&ref=ss_til&asins=B007WTAJTO" style="width:120px;height:240px;" scrolling="no" marginwidth="0" marginheight="0" frameborder="0"></iframe>
{% endraw %}

