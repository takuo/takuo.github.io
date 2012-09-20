---
layout: post
title: nasne の HDDを1TBに換装
date: 2012-07-25 17:22
comments: true
keywords: nasne, PS3, HDD, 換装, 1TB
categories: [ nasne, PS3 ]
---
[先日の記事](/blog/2012/07/21/replacing-hdd-of-nasne/)では詳細な換装方法を書きましたが、サンプルに古くて容量の小さいHDDを利用して実験したまででした。
今回、実際に新しい1TBのHDDを買ってきて実践しましたのでリポートです。

<!-- more -->

買ってきたのは東芝の2.5インチ1TB HDD([MQ01ABD100](http://www.amazon.co.jp/dp/B007XCCTEI/ref=as_li_ss_til?tag=takuojp02-22&camp=1027&creative=7407&linkCode=as4&creativeASIN=B007XCCTEI&adid=1K2S5CSEZ6C3GDTCVFP8&&ref-refURL=http%3A%2F%2Flocalhost%3A4000%2Fblog%2F2012%2F07%2F25%2Freplacing-nasne-hdd-with-1tb%2F) 9.5mm厚)で7,489円でした。 
USB接続のHDDケースは玄人志向の2.5インチ USB3.0接続([GW2.5TL-U3](http://www.amazon.co.jp/dp/B007SQJ7LC/ref=as_li_ss_til?tag=takuojp02-22&camp=1027&creative=7407&linkCode=as4&creativeASIN=B007SQJ7LC&adid=0KWAZQJ2F6T10GB9JCQ6&&ref-refURL=http%3A%2F%2Flocalhost%3A4000%2Fblog%2F2012%2F07%2F25%2Freplacing-nasne-hdd-with-1tb%2F))で980円と大変安いものです。ACアダプタがないものですが一時的にコピーするくらいにしか使わないので平気でしょう。  

手順としては[先日の記事](/blog/2012/07/21/replacing-hdd-of-nasne/)どおりで、

**1. nasne から HDDをはずし、PCに接続して dd を使いイメージを取り出す (数分)**

    # dd if=/dev/sdd of=nasne.img bs=512 count=2623488


**2. xfs 領域をマウントし、PCの適当なところに rsync -vaxXH でコピーする (一瞬)**

    # mount -o ro /dev/sdd3 /mnt
    # rsync -vaxXH /mnt/ /var/tmp/nasne-xfs
    ... (略)
    # umount /mnt


**3. アンマウントしてUSBを外し、新しいHDDに入れ替え、イメージを書き込む (数分)**

    # dd if=nasne.img of=/dev/sdd


**4. パーティションを編集(sdd3を作り直し)し、退避したxfs領域を rsync でコピーする (一瞬)**

    # fdisk /dev/sdd
    ... (略)   
    # mkfs.xfs -f /dev/sdd3
    # mount /dev/sdd3 /mnt
    # rsync -vaxXH /var/tmp/nasne-xfs/ /mnt/
    ... (略)
    # umount /mnt


**5. HDDをPCから外し、nasne に装着して起動確認して組み立てる (数分)**


以上の工程すべてで15分ほどで終わりました。(起動も問題なく一発で行けました。前回はやはりHDDが古過ぎたのかもしれません、超遅いし。)  
xfsを操作するのでどうしてもLinuxが必要になってしまいますが、HDDをまるごとコピーするやりかただと2時間から3時間ほどかかるようなので大幅な短縮と言えます。  

蛇足ですが、nasne に外付けのUSB HDDを装着すると、録画データは先に外付けのHDDに保存されていくようです。  
また、torneから参照するとnasneの空き容量は内蔵と外付けの合計値になっており、区別して保存先を指定するようなことはできないみたいです。

取り出したnasneのHDDが500GBで半端だし使い道がないなーと思いましたが、こないだ買ったPS3のHDDが160GBなのでこれと交換するのがいいかもしれません。今時160GBはゴミ同然と判断できるし切り捨てるにもよいでしょう…。  

{% raw %}
<iframe src="http://rcm-jp.amazon.co.jp/e/cm?lt1=_blank&bc1=000000&IS2=1&bg1=FFFFFF&fc1=000000&lc1=0000FF&t=takuojp02-22&o=9&p=8&l=as4&m=amazon&f=ifr&ref=ss_til&asins=B007V9T9ZK" style="width:120px;height:240px;" scrolling="no" marginwidth="0" marginheight="0" frameborder="0"></iframe>
<iframe src="http://rcm-jp.amazon.co.jp/e/cm?lt1=_blank&bc1=000000&IS2=1&bg1=FFFFFF&fc1=000000&lc1=0000FF&t=takuojp02-22&o=9&p=8&l=as4&m=amazon&f=ifr&ref=ss_til&asins=B007SQJ7LC" style="width:120px;height:240px;" scrolling="no" marginwidth="0" marginheight="0" frameborder="0"></iframe>
<iframe src="http://rcm-jp.amazon.co.jp/e/cm?lt1=_blank&bc1=000000&IS2=1&bg1=FFFFFF&fc1=000000&lc1=0000FF&t=takuojp02-22&o=9&p=8&l=as4&m=amazon&f=ifr&ref=ss_til&asins=B007XCCTEI" style="width:120px;height:240px;" scrolling="no" marginwidth="0" marginheight="0" frameborder="0"></iframe>
{% endraw %}
