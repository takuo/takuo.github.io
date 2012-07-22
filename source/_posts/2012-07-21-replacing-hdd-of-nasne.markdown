---
layout: post
title: nasne のHDDを時間かけずに換装する
date: 2012-07-21 02:00
comments: true
categories: [nasne, PS3]
---
HDDをまるごとバックアップして書き戻す作業は大変時間がかかるので、必要な分だけイメージを取得するようにして待ち時間を短縮します。  
特別な機材は必要ありませんが、USBのHDDがケースがあると便利だと思います。  
Linuxでの作業になるので多少のLinuxの知識が必要になります。  

<!-- more -->

前提として作業はLinux上で行い、使うツールは dd, parted, fdisk, xfsprogs などです。  
nasne の HDD は2.5インチの7mm厚ですが、構造的に換装するHDDは9.5mm厚でも問題ありません。  
※  注意 : **nasneのHDDにはハードウェア固有の情報が書き込まれており、別の個体では利用できません。**

まず nasne から HDD を取り外し、Linux PCに付けます。(USB HDDケースがあると便利) 
nasne の HDD を __/dev/sdd__ として認識したとします。  
xfs の領域である __/dev/sdd3__ をtar (rsync -aHxXとかでも可)でバックアップします。

    # mount -o ro /dev/sdd3 /mnt
    # cd /mnt
    # tar -cpzf /backup/xfs.tgz .
    # cd / && umount /dev/sdd3

パーティション番号2までのサイズ(単位はセクタにする)をメモします。

    # parted /dev/sdd
    GNU Parted 2.3
    Using /dev/sdd
    Welcome to GNU Parted! Type 'help' to view a list of commands.
    (parted) unit s
    (parted) p
    Model: Hitachi HTS545050A7E380 (scsi)
    Disk /dev/sdd: 62500864s
    Sector size (logical/physical): 512B/512B
    Partition Table: msdos
    
    Number  Start     End        Size       Type     File system  Flags
     1      2048s     526335s    524288s    primary  ext3         boot
     2      526336s   2623487s   2097152s   primary  ext3
     3      2623488s  62500863s  59877376s  primary  xfs
    
    (parted) quit

セクタサイズは512Bで、 __/dev/sdd2__ のEndが __2623487__ であることがわかります。  
dd で先頭から __/dev/sdd2__ の終わりまでのイメージを取得します。  
※ セクタ番号は0から始まるのでcountは1増やす。

    # cd / && umount /mnt
    # dd if=/dev/sdd of=/backup/hdd.img bs=512 count=2623488

換装用の新しいディスクを接続します。 __/dev/sde__ とします。 (今回は敢えて小さい120GBのHDDを使用してます)   
ディスクイメージを書き込みます。

    # dd if=/backup/hdd.img of=/dev/sde


または直接 /dev/sdd から /dev/sde に dd してもいいと思います。  
イメージサイズは1GB程度なので大して時間はかかりません。

/dev/sde3 を消して作り直します。

    # fdisk /dev/sde
    
    Command (m for help): p
    
    Disk /dev/sde: 120.0 GB, 120034123776 bytes
    255 heads, 63 sectors/track, 14593 cylinders, total 234441648 sectors
    Units = sectors of 1 * 512 = 512 bytes
    Sector size (logical/physical): 512 bytes / 512 bytes
    I/O size (minimum/optimal): 512 bytes / 512 bytes
    Disk identifier: 0x000d7605
    
       Device Boot      Start         End      Blocks   Id  System
    /dev/sde1   *        2048      526335      262144   83  Linux
    /dev/sde2          526336     2623487     1048576   83  Linux
    /dev/sde3         2623488    62500863    29938688   83  Linux
    
    Command (m for help): d
    Partition number (1-4): 3
    
    Command (m for help): p
    
    Disk /dev/sde: 120.0 GB, 120034123776 bytes
    255 heads, 63 sectors/track, 14593 cylinders, total 234441648 sectors
    Units = sectors of 1 * 512 = 512 bytes
    Sector size (logical/physical): 512 bytes / 512 bytes
    I/O size (minimum/optimal): 512 bytes / 512 bytes
    Disk identifier: 0x000d7605
    
       Device Boot      Start         End      Blocks   Id  System
    /dev/sde1   *        2048      526335      262144   83  Linux
    /dev/sde2          526336     2623487     1048576   83  Linux
    
    Command (m for help): n
    Partition type:
       p   primary (2 primary, 0 extended, 2 free)
       e   extended
    Select (default p): p
    Partition number (1-4, default 3):
    Using default value 3
    First sector (2623488-234441647, default 2623488):
    Using default value 2623488
    Last sector, +sectors or +size{K,M,G} (2623488-234441647, default 234441647):
    Using default value 234441647
    
    Command (m for help): w
    The partition table has been altered!
    
    Calling ioctl() to re-read partition table.
    Syncing disks.
    
xfsファイルシステムを作り直します。  
    
    # mkfs.xfs -f /dev/sde3
    meta-data=/dev/sde3              isize=256    agcount=4, agsize=7244318 blks
             =                       sectsz=512   attr=2, projid32bit=0
    data     =                       bsize=4096   blocks=28977270, imaxpct=25
             =                       sunit=0      swidth=0 blks
    naming   =version 2              bsize=4096   ascii-ci=0
    log      =internal log           bsize=4096   blocks=14149, version=2
             =                       sectsz=512   sunit=0 blks, lazy-count=1
    realtime =none                   extsz=4096   blocks=0, rtextents=0

__/dev/sde3__ をマウントしてtarでバックアップしたxfs領域を展開します。  
(これやらなくても動きましたがやったほうがいい気がします)

    # mount /dev/sde3 /mnt
    # tar -xzpf /backup/xfs.tgz -C /mnt
    # umount /mnt

ディスクを nasne に取り付けて起動します。
{% img /assets/screenshot/nasne-hdd-replace.png %}

よくわかりませんが、起動時に電源LEDとREC LEDが両方赤点灯でフリーズすることがありました。  
なんどか同じ手順をやり直していたら起動したりしなかったりしました。  
サンプルにつかったHDD(HTS541612J9SA00)が古過ぎて調子が悪いのかもしれませんが、原因は定かではありません。  

{% raw %}
<iframe src="http://rcm-jp.amazon.co.jp/e/cm?lt1=_blank&bc1=000000&IS2=1&bg1=FFFFFF&fc1=000000&lc1=0000FF&t=takuojp02-22&o=9&p=8&l=as4&m=amazon&f=ifr&ref=ss_til&asins=B007NTJWB4" style="width:120px;height:240px;" scrolling="no" marginwidth="0" marginheight="0" frameborder="0"></iframe>
<iframe src="http://rcm-jp.amazon.co.jp/e/cm?lt1=_blank&bc1=000000&IS2=1&bg1=FFFFFF&fc1=000000&lc1=0000FF&t=takuojp02-22&o=9&p=8&l=as4&m=amazon&f=ifr&ref=ss_til&asins=B007XCCTEI" style="width:120px;height:240px;" scrolling="no" marginwidth="0" marginheight="0" frameborder="0"></iframe>
<iframe src="http://rcm-jp.amazon.co.jp/e/cm?lt1=_blank&bc1=000000&IS2=1&bg1=FFFFFF&fc1=000000&lc1=0000FF&t=takuojp02-22&o=9&p=8&l=as4&m=amazon&f=ifr&ref=ss_til&asins=B004QZAPMS" style="width:120px;height:240px;" scrolling="no" marginwidth="0" marginheight="0" frameborder="0"></iframe>
<iframe src="http://rcm-jp.amazon.co.jp/e/cm?lt1=_blank&bc1=000000&IS2=1&bg1=FFFFFF&fc1=000000&lc1=0000FF&t=takuojp02-22&o=9&p=8&l=as4&m=amazon&f=ifr&ref=ss_til&asins=B0053VPYHK" style="width:120px;height:240px;" scrolling="no" marginwidth="0" marginheight="0" frameborder="0"></iframe>
<iframe src="http://rcm-jp.amazon.co.jp/e/cm?lt1=_blank&bc1=000000&IS2=1&bg1=FFFFFF&fc1=000000&lc1=0000FF&t=takuojp02-22&o=9&p=8&l=as4&m=amazon&f=ifr&ref=ss_til&asins=B007V9T9ZK" style="width:120px;height:240px;" scrolling="no" marginwidth="0" marginheight="0" frameborder="0"></iframe>
<iframe src="http://rcm-jp.amazon.co.jp/e/cm?lt1=_blank&bc1=000000&IS2=1&bg1=FFFFFF&fc1=000000&lc1=0000FF&t=takuojp02-22&o=9&p=8&l=as4&m=amazon&f=ifr&ref=ss_til&asins=B0035V5EA2" style="width:120px;height:240px;" scrolling="no" marginwidth="0" marginheight="0" frameborder="0"></iframe>
{% endraw %}
