---
layout: post
title: "CyanogenMod で DTCP-IP やパズドラを動かす"
date: 2013-07-11 23:49
comments: true
categories: [ "Android", "CyanogenMod", "root" ]
---

CyanogenMod でもパズドラを起動したいとです！！！  

<!-- more -->

というわけなんですが、パズドラとかどうでもよくて TwonkyBeam の DTCP-IP を使いたいというが目的で調べました。  
要するに root チェックにより CyanogenMod のようなカスタムROMでは使用できないアプリを動かすというやつです。


インターネットで調べると以下のような項目でroot端末かどうかをチェックしているようです。

* **su バイナリの存在** /system/{bin,xbin}, /sbin あたりにsu があるとダメなようです
* **/system/app/Superuser.apk の存在** (CyanogenModにはないですがリネームすればいいだけです)
* **getprop *ro.secure* の値が0だとNG** (CyanogenModでは1なので問題ないです)
* **getprop *ro.debuggable* の値が1だとNG** (CyanogenModでは1なので対策が必要です)

**su のバイナリ** は */system/bin/su*, */system/xbin/su* にあるので適当にリネームして回避出来ますが、リネームしっぱなしだと root が必要なアプリを実行するときにこまるのでこんなスクリプトを書いてみました。  

まずシステムの起動時に実行される _/data/local/userinit.sh_ で **su** を **sux** にリネームして隠します。
``` bash /data/local/userinit.sh
#!/system/bin/sh

mount -o remount,rw /system

#
# su を退避 symlinkの場合は無視
if [ -x /system/bin/su -a ! -L /system/bin/su ]; then
    mv /system/bin/su /system/bin/sux
fi
if [ -x /system/xbin/su -a ! -L /system/xbin/su ]; then
    mv /system/xbin/su /system/xbin/sux
fi

mount -o remount,ro /system
```

次に su があれば削除し、なければ sux から symlink を貼るというようなスクリプトです。
``` bash /data/local/swsu.sh
#!/system/bin/sh

# rootじゃない場合sux使ってrootになる
if [ `whoami` != "root" ]; then
  sux -c $0
  exit
fi

mount -o remount,rw /system

if [ ! -L /system/xbin/su ]; then
  ln -s /system/xbin/sux /system/xbin/su
  ln -s /system/bin/sux /system/bin/su
  echo "enable su"
else
  rm -f /system/xbin/su
  rm -f /system/bin/su
  echo "disable su"
fi

mount -o remount,ro /system
```

どちらも配置したら **chmod 0755** するのを忘れずに。  

ひとまずこれで再起動してみて試します。  
__/data/local/swsu.sh__ は [CommandRunner](https://play.google.com/store/apps/details?id=com.ryo.commandRunner) とかで起動したら良いでしょう。  
ショートカットも作れるので便利です。  
実行してみて su の存在をトグルできるようになってればOKです。  
※ root をトグルするアプリも存在しますが、挙動を把握しやすくするため今回は自分でスクリプト書きました。  

さて、次に **getprop** の値。まず **/default.prop** の値が云々的な説明をしているサイトがありますが間違いです。  
目的のプロパティは **boot.img** の中、 **initrd.img** にある **default.prop** により設定されてます。(んでその initrd が / にマウントされる)  
そんなわけで **boot.img** をいじる必要があります。結構めんどいです。  

まず **boot.img** を *update.zip* から取り出しておきます。  
**boot.img** の展開と再構築には Linux上で *abootimg* を使いますが、WindowsやMacでもなんとかする方法はあると思いますので頑張って。  

``` bash terminal
$ sudo apt-get install abootimg unzip
$ wget http://get.cm/get/jenkins/34073/cm-10.1.1-grouper.zip
$ mkdir bootimg
$ cd bootimg
$ unzip ../cm-10.1.1-grouper.zip boot.img
$ abootimg -x boot.img
$ mkdir initrd
$ cd initrd
$ zcat ../initrd.img | cpio -i
$ sed -e 's/ro.debuggable=1/ro.debuggable=0/g' default.prop > default.prop.new
$ mv default.prop.new default.prop
$ find . | cpio -o -H newq | gzip > ../initrd.img
$ cd ..
$ abootimg --create newboot.img -k zImage -r initrd.img
```

で、**newboot.img** ができたらひとまず *fastboot* から起動してみます。  
``` bash terminal
$ adb reboot bootloader
$ sudo fastboot boot newboot.img
$ adb shell

shell@android:/ $ getprop ro.debuggable
0
shell@android:/ $
```

これでパズドラは起動しますし、TwonkyBeam で DTCP-IP が使えるようになりました。  
nasneで録画した動画やライブチューナでテレビも見られます！   

しかしまだ終わりません。先ほどの **swsu.sh** を実行してみます。  
動きませんね。  
この状態だと su (sux) が動かないことに気が付きます。  
なぜか確認なしにsuが拒否されてしまうのです。
logcat には次のように出力されています。
``` text terminal
W/su      ( 8151): request rejected (10097->0 /system/bin/sh) F8?HO? 
D/su      ( 8147): su invoked. 
D/su      ( 8149): access_disabled 
E/su      ( 8148): Root access is disabled on non-debug builds
```

なるほど。  
実は CyanogenMod で提供している su は変更が加えてあり、 *ro.debuggable=0* では無条件に拒否するようになっています。  

しかし、[ソース](https://github.com/CyanogenMod/Superuser/blob/379371bcff3f18b4a16beb37a35c2b211a941781/Superuser/jni/su/su.c#L548)をみてみると、runtime に **/default.prop** を見ているだけのようなので、システムが起動した後に **/default.prop** を書き換えてやればよさそうです。  
でも、あれ？？？  
でも起動したら su 使えないし書き換えられないんじゃ？  
はい。そうです。そこで先ほどの **userinit.sh** に追記します。 **userinit.sh** は root権限で動きますからね。  

普通に再起動して以下を**追記**した **userinit.sh** を改めて **/data/local/userinit.sh** へ置きます。 **chmod 0755** を忘れずに。(recovery で adb が使えれば recovery で作業してもいいです)  

``` bash /data/local/userinit.sh(抜粋)
#
# /default.prop書換え
mount -o remount,rw /
sed -e 's/ro.debuggable=0/ro.debuggable=1/g' /default.prop > /default.prop.tmp
mv /default.prop.tmp /default.prop
chmod 0644 /default.prop
mount -o remount,ro /
```

これでもう一度 fastboot boot newboot.img して確認してみましょう。  

``` bash terminal
$ adb reboot bootloader
$ sudo fastboot boot newboot.img
$ adb shell
shel@android:/ $ getprop ro.debuggable
0
shell@android:/ $ sux
shell@android:/ #
```

うまく行きました。  
満を持して newboot.img を fastboot で flash しましょう。
``` bash terminal
$ adb reboot bootloader
$ sudo fastboot flash boot newboot.img
```

そんなわけで、/data/local/ は wipe しない限り維持できますから、今後はアップデートするときに boot.img を改変するだけでよくなります。  
boot.img の改変は面倒ですが、手順は決まっているしスクリプトにすれば自動化はできるかと思います。  
(というか前述の手順がスクリプトそのものっぽい)

