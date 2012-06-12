---
layout: post
title: "TwiProwlをAWS EC2で動かす(無料)"
date: 2012-06-12 19:37
comments: true
categories: [AWS, TwiProwl]
---
{% img right https://lh5.googleusercontent.com/-uzJ0rqEN72U/T9cintiTKvI/AAAAAAAAIqI/W8owjZxM-QE/s800/aws-logo.png http://aws.amazon.com/jp/ %}
[AWS](http://aws.amazon.com/jp/)のEC2 Linux マイクロインスタンスは無料範囲をキープしていれば１年間[無料](http://aws.amazon.com/jp/free/)で使え、[TwiProwl](http://github.com/takuo/TwiProwl)いっこ動かすくらいなら余裕でキープできそうなのでやってみます。


※ TwiProwlは拙作のTwitter通知ツールですが説明は割愛します。
<!-- more -->

まず、インスタンスの作成ですが、サービスコンソールから"Launch Instance"を選んで、"Quick Launch Wizard"、"Ubuntu Server 11.10 (64bit)"を選び次へ進みます。  
ほかのLinuxインスタンスでも出来なくないでしょうが今回はこれで行きます。ちなみに12.04 LTSだとKernelがおかしいらしくProcess#daemonがまともに動かなかったのでやめたほうがいいです。(linux-image-3.2.0-24-virtual)  
{% img https://lh5.googleusercontent.com/-ERrv6R2Zzlg/T9chQmozptI/AAAAAAAAIpo/Qn6I95WKHf0/s400/aws-launch-instance.png https://lh5.googleusercontent.com/-ERrv6R2Zzlg/T9chQmozptI/AAAAAAAAIpo/Qn6I95WKHf0/s800/aws-launch-instance.png %}

詳細設定はデフォルトのまま。  
{% img https://lh3.googleusercontent.com/-b3UKWibIm_w/T9chQrjhOEI/AAAAAAAAIpc/ijO-B4uFCag/s400/create-new-instance.png https://lh3.googleusercontent.com/-b3UKWibIm_w/T9chQrjhOEI/AAAAAAAAIpc/ijO-B4uFCag/s800/create-new-instance.png %}

起動しました。  
{% img https://lh4.googleusercontent.com/-aOuzv5tYklI/T9chWfDzg_I/AAAAAAAAIp0/M_1gq2PojGo/s400/instance-status.png https://lh4.googleusercontent.com/-aOuzv5tYklI/T9chWfDzg_I/AAAAAAAAIp0/M_1gq2PojGo/s800/instance-status.png %}

sshでのログイン方法もコンソールのInstance ActionからConnectを選べば説明がでてくるのでその通りにやればよろしいのですが、rootではログインできないのでubuntuユーザでログインすることになります。  
{% img https://lh3.googleusercontent.com/-NfQGIxuBO3A/T9chQnhlknI/AAAAAAAAIpk/R8-bFy7eBsY/s400/connect-to-instance.png https://lh3.googleusercontent.com/-NfQGIxuBO3A/T9chQnhlknI/AAAAAAAAIpk/R8-bFy7eBsY/s800/connect-to-instance.png %}

インスタンスを起動してsshでログインできたら実行環境をセットアップします。

    $ sudo -s
    # apt-get update && apt-get upgrade
    # apt-get install ruby1.9.1 git
    # gem install oauth ruby-hmac
    # exit
    $ git clone http://github.com/takuo/TwiProwl.git

設定ファイルを適当に書きます。(~/.twiprowl.conf として保存)

``` yaml /home/ubuntu/.twiprowl.conf
    LogDir: /home/ubuntu/var/log
    Debug: false
    Daemon: true

    Prowl:
      APIKey: ProwlのAPIキーを書く

    NMA:
      APIKey: NMAを使う場合、NMAのAPIキーを書く

    Accounts:
     -
      Application: "Twitter"
      User: screen_name
      Enable: true
      NotifyMethods:
       - "prowl"               # Prowl for iPhone
       - "nma"                 # Notify My Android for Android
      RateLimitThreshold: 20
      Mentions:                # Event on Mentions
       Priority: 1
       IgnoreSelf: false       # Ignore mention by myself
      Direct:                  # Event on Direct Messages
       Priority: 1
       IgnoreSelf: false       # Ignore DM event from myself
      Retweets:                # Event on Retweets
       Priority: 0
      Membership:              # Event on List Membership
       Priority: 0
       IgnoreSelf: false       # Ignore event by myself
       IgnoreNegative: false   # Ignore remove event
      Favorite:                # Event on Favorite/Unfavorite
       Priority: 0
       IgnoreSelf: false       # Ignore event by myself
       IgnoreNegative: false   # Ignore Unfavorite event
      Unfollowed:              # Event on Unfollowed, requiresi API rate
       Priority: 0
       Interval: 60
```

適当です。これ以上のことは自分で調べて好きなようにしたらいいと思います。
動かす。

    $ mkdir -p ~/var/log
    $ cd ~/TwiProwl
    $ ./twiprowl
    oauthなんたらかんたら(認証)

おわり。  
余計なもの(ruby1.8とか)を入れた場合の不都合は知りません。  
無料範囲に収まらず課金が発生しても責任は負いません。  
今の所1年間ということらしいので1年後は気を付けましょう。
{% raw %}
<iframe src="http://rcm-jp.amazon.co.jp/e/cm?lt1=_blank&bc1=000000&IS2=1&bg1=FFFFFF&fc1=000000&lc1=0000FF&t=takuojp02-22&o=9&p=8&l=as4&m=amazon&f=ifr&ref=ss_til&asins=4844329804" style="width:120px;height:240px;" scrolling="no" marginwidth="0" marginheight="0" frameborder="0"></iframe>
{% endraw %}
