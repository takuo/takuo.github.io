---
layout: post
title: "ffmpeg で ustream 配信したメモ"
date: 2013-05-30 17:26
comments: true
categories: Linux
---
どうも。

当方、[Ustreamで猫カメラ](http://ustream.tv/channel/northeye)を運用しているのだけれど、FlashMediaEncoderを動かしていたWindowsがとても不安定で廃絶してしまったので、どうしようかと思っていました。  
それと、常時稼動マシンは減らしたい。  
そんなわけでいつも動いているNASマシンかルーターマシンで動かすことにしました。  

<!-- more -->

NASやルーターはだいたいLinuxでやってるのでLinuxでやることになります。  
KVMにWindows環境つくってPCIパススルーでUSBデバイスをスルーして配信することも可能ですが、エンコードというプロセスが存在するため、ある程度のパフォーマンスが必要になり、仮想環境ではなかなか無理がありました。(試したけどやめた)  

そんなわけで、今回はルーターにffmpegいれてこいつでやってみます。  
ちなみにカメラデバイスはLogicoolのC910というHDウェブカメラで、ステレオマイクがついています。  
※ もう売ってなくて[C920](http://amzn.to/11dbZvi)という機種が後継のようです。  

## lxcで環境構築

しかしながら、ルーターのような重要なマシンに余計なものを入れたくないのも事実。  
そこでlxcのコンテナで環境を分離します。あとで別のマシンでやりたくなったとき移行も簡単なので便利です。  
lxcの環境構築は省略して、ポイントだけ。  

lxcの設定ファイルにv4l2とsndのcgroupパーミッションを追加します。

    # v4l2
    lxc.cgroup.devices.allow = c 81:* rwm
    # sound
    lxc.cgroup.devices.allow = c 116:* rwm

ホスト側の/dev/sndをbind mountしときます。

    % sudo mount --bind /dev/snd /lxc/ustream/rootfs/dev/snd

lxcをstartします。

    % sudo lxc-start -n ustream -d


以降はlxc環境での操作になります。  
ffmpegはubuntuやdebianのだとlibavになってるのでdeb-multimedia.orgから入れます。

    # echo "deb http://www.deb-multimedia.org sid main non-free" > /etc/apt/sources.list.d/multimedia.list
    # echo 'APT::Install-Recommends "0";' > /etc/apt/apt.conf.d/99norecommends 
    # apt-get update
    # apt-get install ffmpeg alsa-utils

自分をaudioとvideoのグループに追加して、/dev/video0 や /dev/snd あたりのパーミッション確認をしときましょう。  
カメラは単にUSBカメラさすだけで /dev/video0 は参照できました。  
audioの方ですがひとまず /proc/asound あたりを見てどういう状態か確認します。pulse audioはよく知りません。

    % cat /proc/asound/cards
     0 [PCH            ]: HDA-Intel - HDA Intel PCH
                          HDA Intel PCH at 0xfe700000 irq 63
     1 [U0x46d0x821    ]: USB-Audio - USB Device 0x46d:0x821
                          USB Device 0x46d:0x821 at usb-0000:00:1d.0-1.7.4, high speed
    % cat /proc/asound/devices
      2: [ 0- 3]: digital audio playback
      3: [ 0- 2]: digital audio capture
      4: [ 0- 1]: digital audio playback
      5: [ 0- 0]: digital audio playback
      6: [ 0- 0]: digital audio capture
      7: [ 0- 3]: hardware dependent
      8: [ 0- 2]: hardware dependent
      9: [ 0]   : control
     10: [ 1- 0]: digital audio capture
     11: [ 1]   : control
     33:        : timer

というわけで、card  1がUSBカメラのマイクっぽくて、1-0がキャプチャのようなので ffmpeg で渡すオプションには hw:1,0で良さげです。

## Ustreamの配信用RTMP URL

ustreamの配信先URLを確認します。  
ログインしてダッシュボード→番組設定→リモート のところに RTMP URLとストリームキーがあるので、結合してひとつのURLにします。
( URL + "/" + キー )

## ffmpeg の実行

ffmpeg を実行してみます。

    % ffmpeg -f v4l2 -i /dev/video0 -ac 2 -f alsa -i hw:1,0 -c:a libfaac -ar 22050 -b:a 48k -c:v libx264 -preset veryfast -threads auto -vsync 1 -f flv 'rtmp://1.xxx.fme.ustream.tv/ustreamVideo/xxxxxx/<キー> flashver=FME/3.0\20(compatible;\20FMSc/1.0)'

とまぁこんな感じです。  
x264のプリセットはveryfastとかにしたほうが遅延が少なくてすむのかなと思いました。

## TIPS

とりあえず ffmpeg のオプションの順番には意味があるので注意しましょう。  

### マイクのミュートとかを動的にやりたい。

ffmpegを止めずにマイクのオンオフはできたほうがいいですね。

    % alsamixer -c 1 

してF4でキャプチャのミキサーにしてSPACEキーでトグル。

### iOSやAndroidのアプリで視聴できるフォーマットにしたい

いろいろ調べましたが、baseline profileでLevel3.0ならいいらしいです。  
対応するffmpegのオプションは

    -profile:v baseline -level 3.0 

ということなのですが、ただこれをつけただけだと

    x264 [error]: baseline profile doesn't support 4:2:2
    [libx264 @ 0xb7fa40] Error setting profile baseline.

となり失敗します。  
どうやらpixel formatがよくないようで、さらに

    -pix_fmt yuv420p

というオプションをつければ良いようです。

### たまにffmpegが死ぬ

Ustreamサーバの調子にも左右されますがごく稀にffmpegプロセスが死んでますので。

    % while true; do
    ffmpeg -r 15 -s 640x360 -f v4l2 -i /dev/video0 -ac 2 -f alsa -i hw:1,0 -acodec libfaac -ar 22050 -b:a 48k -c:v libx264 -profile:v baseline -level 3.0 -preset veryfast -pix_fmt yuv420p -threads 3 -vsync 1 -y -f flv 'rtmp://1.xxxx.fme.ustream.tv/ustreamVideo/streamid/streamkey flashver=FME/3.0\20(compatible;\20FMSc/1.0)'
    sleep 5
    done

みたいに実行するようにしました。

### オーディオのキャプチャができない

当方で遭遇したのはマイクのチャンネル数を合わせないといけないという事でした。  
マイクがモノラルなら -ac 1 、ステレオなら -ac 2 としないといけないようです。

### ホワイトバランスなどの調整

v4l2-ctl -c でやります。  
v4l2-ctl -L で何ができるかわかります。  

    % v4l2-ctl -L 
                         brightness (int)    : min=0 max=255 step=1 default=128 value=128
                           contrast (int)    : min=0 max=255 step=1 default=32 value=32
                         saturation (int)    : min=0 max=255 step=1 default=32 value=32
     white_balance_temperature_auto (bool)   : default=1 value=0
                               gain (int)    : min=0 max=255 step=1 default=64 value=57
               power_line_frequency (menu)   : min=0 max=2 default=2 value=2
    				0: Disabled
    				1: 50 Hz
    				2: 60 Hz
          white_balance_temperature (int)    : min=2800 max=6500 step=1 default=4000 value=4300
                          sharpness (int)    : min=0 max=255 step=1 default=72 value=72
             backlight_compensation (int)    : min=0 max=1 step=1 default=0 value=0
                      exposure_auto (menu)   : min=0 max=3 default=3 value=3
    				1: Manual Mode
    				3: Aperture Priority Mode
                  exposure_absolute (int)    : min=3 max=2047 step=1 default=166 value=415 flags=inactive
             exposure_auto_priority (bool)   : default=0 value=1
                       pan_absolute (int)    : min=-36000 max=36000 step=3600 default=0 value=0
                      tilt_absolute (int)    : min=-36000 max=36000 step=3600 default=0 value=0
                     focus_absolute (int)    : min=0 max=255 step=17 default=68 value=0
                         focus_auto (bool)   : default=1 value=0
                      zoom_absolute (int)    : min=1 max=5 step=1 default=1 value=1

    % v4l2-ctl -c white_balance_temperature=4300

ffmpeg 実行するたびにリセットされる気がする。


{% raw %}
<iframe src="http://rcm-jp.amazon.co.jp/e/cm?lt1=_blank&bc1=000000&IS2=1&bg1=FFFFFF&fc1=000000&lc1=0000FF&t=takuojp02-22&o=9&p=8&l=as4&m=amazon&f=ifr&ref=ss_til&asins=B006NTPLJW" style="width:120px;height:240px;" scrolling="no" marginwidth="0" marginheight="0" frameborder="0"></iframe>
<iframe src="http://rcm-jp.amazon.co.jp/e/cm?lt1=_blank&bc1=000000&IS2=1&bg1=FFFFFF&fc1=000000&lc1=0000FF&t=takuojp02-22&o=9&p=8&l=as4&m=amazon&f=ifr&ref=ss_til&asins=B001VA2L4Q" style="width:120px;height:240px;" scrolling="no" marginwidth="0" marginheight="0" frameborder="0"></iframe>

{% endraw %}

