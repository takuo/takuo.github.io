---
layout: post
title: "Xfce(など)でChromeのプロキシ設定をする"
date: 2012-04-18 11:12
comments: true
categories: misc
---

Xfceなど、システムのプロキシ設定ができない環境だとChromeはプロキシの設定が利用できず、次のように表示されてしまいます。  
{% img http://ga.vg/g/0238.png %}

コマンドラインからは設定できるとあるので、やってみます。  

<!-- more -->

まず *proxy.pac* ファイルを書きます。 
書き方については[ぐぐれ](https://www.google.co.jp/search?q=proxy.pac)ばいくらでも出てきますが、例えばこんな感じに。

``` javascript /home/user/proxy.pac
function FindProxyForURL(url,host)
{
   if(isPlainHostName(host)||
      isInNet(host,"192.168.0.0","255.255.0.0") ||
      isInNet(host, "127.0.0.0" , "255.0.0.0") ||
      shExpMatch( host, "*.example.jp") ||
      host = "localhost"
     )
     return "DIRECT";
   else
     return "PROXY 192.168.0.1:3128";
}
```


次に、デスクトップやメニュー、パネルのランチャー設定をかえます。  
*/usr/share/applications/google-chrome.desktop* の

    Exec=/opt/google/chrome/google-chrome %U

のようになっている部分を

    Exec=/opt/google/chrome/google-chrome --proxy-pac-url=file:///home/user/proxy.pac %U

にして、*~/.local/share/applications/google-chrome.desktop* にコピー。  
以上で、プロキシの設定として */home/user/proxy.pac* が使われるようになりました。 

気をつけることは、chromeが立ち上がってない状態で別のアプリケーションからの起動の際、これだけではカバーできないことです。  
そういう場合はラッパースクリプトでも用意しておきましょう。

``` sh /opt/google/chrome/google-chrome
#!/bin/sh

exec /opt/google/chrome/google-chrome.real \
  --proxy-pac-url=file:///home/user/proxy.pac $*
```
