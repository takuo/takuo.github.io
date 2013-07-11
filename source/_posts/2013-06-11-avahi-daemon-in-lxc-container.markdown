---
layout: post
title: "LXC コンテナで avahi-daemon を動かす"
date: 2013-06-11 16:41
comments: true
categories: [ Linux, LXC ]
---
LXCコンテナが役割ごとに増殖してて、DHCPつかってアドレス振ってると、振られたIPアドレス調べたり色々面倒なのでmDNSでアレしたいなと。  

というわけで mDNS であるところの **avahi-daemon** を動かそうとしたんですが動きませんのでメモ。  

<!-- more -->

調べによるとLXCのコンテナでは **setrlimit(2)** システムコールが使えないらしいです。  
**avahi-daemon** はデフォルトでこのシステムコールを使うので起動しません。  
で、オプションに *--no-rlimits* をつけて起動すれば良いとのこと。  

Ubuntuだとこのへん (/etc/init/avahi-daemon.conf) の opts に *--no-rlimits* を追加
``` bash /etc/init/avahi-daemon.conf
# avahi-daemon - mDNS/DNS-SD daemon
#
# The Avahi daemon provides mDNS/DNS-SD discovery support (Bonjour/Zeroconf)
# allowing applications to discover services on the network.

description     "mDNS/DNS-SD daemon"

start on (filesystem
          and started dbus)
stop on stopping dbus

expect daemon
respawn

pre-start script
    /lib/init/apparmor-profile-load usr.sbin.avahi-daemon
end script

script
        opts="-D --no-rlimits"
        [ -e "/etc/eucalyptus/avahi-daemon.conf" ] && opts="${opts} -f /etc/eucalyptus/avahi-daemon.conf"
        exec avahi-daemon ${opts}
end script
```

これで動くので *hostname.local* といった名前でコンテナにアクセスできるようになりました。  
*--no-rlimits* でどういう影響がでるのかは調べていません。  

