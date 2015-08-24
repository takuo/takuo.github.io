---
layout: post
title: "Qualcomm IZat のメモリリーク"
date: 2015-08-24 23:40
comments: true
categories: [Android]
---
{% img right /assets/screenshot/QualcommIZat.png /assets/screenshot/QualcommIZat.png 240 230 %}
QualcommのSoCにIZatというテクノロジーがあって、これは位置情報の効率化を図るハードウェア支援だそうです。  
au版のXperia Z3ではこれを有効化できるのですが、どうやらメモリリークが存在するようでした。  

<!-- more -->
IZatを有効化するとlowi-serverというのが働き出してどんどんメモリを食っていきます。  
{% img /assets/screenshot/lowi-server.png 640 601 %}
こんな感じです。  
6日間で366MBになっています。

この状態だとあらゆるバックグラウンドアクティビティはシステムによってkillされまくり、ウェブブラウザもホームに戻る度にロードし直しで不快極まりないです。  
通知を握りつぶされることもままあります。  

しかもこいつはシステムプロセスなので、root権がなくてはkillすることもできず、こうなってしまったら端末を再起動するしかありません。  

ちなみにlogcatしているとなにやら延々とエラーがでていますがよくわかりません。  
``` text logcat
W/LOWI-SERVER(  485): [LOWIController] Error in getting response from cache
W/LOWI-SERVER(  485): [LOWIScanResultReceiver] No result, due to terminate or no memory
```

というわけで、こんなもんは使わないほうが良いです。  
さっさとOFFにして再起動しましょう。(ちなみにデフォルトOFFです)  
OFFであれば0.3MB程度しか消費しません。  

docomoやソフトバンクのZ3ではそもそも有効化できないので心配いりません。Z4は知りませんが。  

現場からは以上です。

