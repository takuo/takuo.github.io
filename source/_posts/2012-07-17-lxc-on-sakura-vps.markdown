---
layout: post
title: "さくらのVPSでLXC(Linux Container)を動かす"
date: 2012-07-17 19:32
comments: true
categories: [vps, lxc]
---
yaegashiセムパイが"[さくらのVPSでOpenVZを動かす](http://blog.keshi.org/hogememo/2012/04/05/openvz-on-sakura-vps)"というエントリを書かれていて、自分もやろうと思ったのですが、OpenVZはカーネルに手を入れる必要があり、Debian wheezy環境では面倒だったので代わりに[LXC](http://lxc.sf.net/)という仮想環境を導入することにしました。

<!-- more -->
OpenVZもそうですが、LXCはコンテナ型の仮想環境であり、KVMなどのハードウェア仮想化とは違って仮想化によるオーバーヘッドはほぼありません。  
雑に言えばchrootなだけなので、VT-xのような仮想化支援付きのCPUすら不要で、KVMやVMware,VirtualBoxといった仮想環境と併用できます。

### VPSで仮想環境を構築する理由
[さくらのVPS](http://vps.sakura.ad.jp/)はKVM上に構築されているとのことですが、こういった仮想環境で更に仮想環境を動かす意味はなんなのか。  

僕は今のVPS(2Gプラン)に至るまで、3回ほど環境の移行をしてきました。  
1回目は自宅サーバからさくらVPS(512)、2回目は1Gプランへ移行、3回目は現在の2Gプランといった具合です。  

そのたび行う移行作業に大変苦労しており、その間ロストされている何かが必ずあって今現在、最初の自宅サーバと全く同一の環境にはなっていないと思います。  

その点仮想環境であれば、いくつかの設定ファイルとrootfsを固めたもを新しい環境で展開し起動するだけで楽かつ完全な移行ができるという事になります。  
また、LXCはLinux kernel標準の機能(cgroupとkernel namespaceを使用)であり、いまどきのLinux環境ならどこでも動かせるOSレベルの仮想化技術です。(OpenVZはpatchがいるのでそうはいきません)  

そんなわけで、今後さくらVPSを離れるか、別のプランに移行するかはわかりませんが、将来に備えVPS上での仮想環境を構築しました。  

### 導入
当方の環境はDebian wheezy(amd64)ですので、それ前提で書きます。

lxcvm00 という名前のコンテナをdebianテンプレートで作成します。
ネットワークはブリッジで、ブリッジインタフェースはlxcbr0を使います。

    host # apt-get install lxc debootstrap
    host # cat <<EOL >> /etc/network/interfaces
    auto lxcbr0
    iface lxcbr0 inet static
            address 10.0.0.1
            netmask 255.255.255.0
            bridge_ports none
            bridge_stp off
    EOL
    host # ifup lxcbr0
    host # echo "cgroup	/sys/fs/cgroup	cgroup	defaults 0 0" >> /etc/fstab
    host # mount cgroup
    host # lxc-create -t debian -n lxcvm00

いくつかdebconfベースで設定を促されるので適当に設定します。
lxc-createが終わったら設定ファイルを修正します。

lxc.devttydir(設定しないと/dev/tty*が作られなかった(なぜ?)), lxc.tty(ttyの数), lxc.cap.drop からsys_adminを除くといったあたりの修正です。

``` text "/var/lib/lxc/lxcvm00/config"
# /var/lib/lxc/lxcvm00/config

## Container
lxc.utsname                             = lxcvm00
lxc.rootfs                              = /var/lib/lxc/lxcvm00/rootfs
lxc.arch                                = x86_64
#lxc.console                            = /var/log/lxc/lxcvm00.console
lxc.devttydir                           = lxc
lxc.tty                                 = 4
lxc.pts                                 = 1024

## Capabilities
lxc.cap.drop                            = mac_admin
lxc.cap.drop                            = mac_override
# lxc.cap.drop                            = sys_admin
lxc.cap.drop                            = sys_module
## Devices
# Allow all devices
#lxc.cgroup.devices.allow               = a
# Deny all devices
lxc.cgroup.devices.deny                 = a
# Allow to mknod all devices (but not using them)
lxc.cgroup.devices.allow                = c *:* m
lxc.cgroup.devices.allow                = b *:* m

# /dev/console
lxc.cgroup.devices.allow                = c 5:1 rwm
# /dev/fuse
lxc.cgroup.devices.allow                = c 10:229 rwm
# /dev/null
lxc.cgroup.devices.allow                = c 1:3 rwm
# /dev/ptmx
lxc.cgroup.devices.allow                = c 5:2 rwm
# /dev/pts/*
lxc.cgroup.devices.allow                = c 136:* rwm
# /dev/random
lxc.cgroup.devices.allow                = c 1:8 rwm
# /dev/rtc
lxc.cgroup.devices.allow                = c 254:0 rwm
# /dev/tty
lxc.cgroup.devices.allow                = c 5:0 rwm
# /dev/urandom
lxc.cgroup.devices.allow                = c 1:9 rwm
# /dev/zero
lxc.cgroup.devices.allow                = c 1:5 rwm

## Limits
#lxc.cgroup.cpu.shares                  = 1024
#lxc.cgroup.cpuset.cpus                 = 0
#lxc.cgroup.memory.limit_in_bytes       = 256M
#lxc.cgroup.memory.memsw.limit_in_bytes = 1G

## Filesystem
lxc.mount.entry                         = proc /var/lib/lxc/lxcvm00/rootfs/proc proc nodev,noexec,nosuid 0 0
lxc.mount.entry                         = sysfs /var/lib/lxc/lxcvm00/rootfs/sys sysfs defaults,ro 0 0
#lxc.mount.entry                        = /srv/lxcvm00 /var/lib/lxc/lxcvm00/rootfs/srv/lxcvm00 none defaults,bind 0 0

## Network
lxc.network.type                        = veth
lxc.network.flags                       = up
lxc.network.hwaddr                      = 00:FF:00:00:00:01
lxc.network.link                        = lxcbr0
lxc.network.veth.pair                   = lxcvm00eth0
```

コンテナ側の/etc/inittabと/etc/securettyを修正します
consoleにgettyを待機させておくと lxc-start でログインプロンプトが出せます(が、lxc-startは -d でバックグラウンドに移したほうがいいと思います。)

``` text "/etc/inittab 抜粋"

c0:2345:respawn:/sbin/getty 38400 console
1:2345:respawn:/sbin/getty 38400 tty1 linux
2:23:respawn:/sbin/getty 38400 tty2 linux
3:23:respawn:/sbin/getty 38400 tty3 linux

```

securettyにはrootでログイン可能なttyにlxc/*を追加してます。
``` text "/etc/securetty 抜粋"

lxc/console
lxc/tty1
lxc/tty2
lxc/tty3

```

ここまでで、最低限起動するまでの準備ができました。


### 起動
lxc-startでコンテナを起動し、lxc-console でconsoleを開きます。

    host # lxc-start -n lxcvm00 -d
    host # lxc-console -n lxcvm00
    Type <Ctrl+a q> to exit the console, <Ctrl+a Ctrl+a> to enter Ctrl+a itself

    Debian GNU/Linux wheezy/sid lxcvm00 tty1

    lxcvm00 login: 

コンソールは Ctrl+a q で抜けられます。
あとはコンテナ内は好きにいじることができます。
/etc/network/interfacesあたりも確認しておきましょう。

``` text "/etc/network/interfaces"
# interfaces(5) file used by ifup(8) and ifdown(8)
auto lo
iface lo inet loopback

auto eth0
iface eth0 inet static
        address 10.0.0.4
        broadcast 10.0.0.255
        netmask 255.255.255.0
        gateway 10.0.0.1
```

### 外からアクセスするための設定

ホストとコンテナ間の通信はブリッジにより行なっていますが、VPSの外、つまりインターネットからのアクセスやインターネットへのアクセスをするためにNATの設定をします。  
外へはeth0にマスカレードし、80と443はコンテナにDNATすることにします。  
sshは不正アクセスが多いので22ではなく適当なポート番号で受け、コンテナの22に渡すことにします。

    host # sysctl -w net/ipv4/ip_forward=1
    host # iptables -t nat -I POSTROUTING -o eth0 -j MASQUERADE
    host # iptables -t nat -I PREROUTING -i eth0 -p tcp -m tcp -m multiport --dports 80,443 -j DNAT --to-destination 10.0.0.4
    host # iptables -t nat -I PREROUTING -i eth0 -p tcp -m tcp --dport 43276 -j DNAT --to-destination 10.0.0.4:22

これで、コンテナ内でHTTPサービスを動かせば外からのHTTP/HTTPSアクセスはコンテナ内で捌くようになります。  
sshはVPSの43276番でコンテナの22にフォワードしてます。

### まとめ
以上、LXCはとくに苦労もなく導入することができるのでオススメです。  
また運用するサービスごとにコンテナを分けておけば、将来httpだけ別の環境に移したいなどといった欲求にも容易に対応できるでしょう。  

次回はリソース制限や動的な変更などを書いてみたいと思います。

{% raw %}
<iframe src="http://rcm-jp.amazon.co.jp/e/cm?lt1=_blank&bc1=000000&IS2=1&bg1=FFFFFF&fc1=000000&lc1=0000FF&t=takuojp02-22&o=9&p=8&l=as4&m=amazon&f=ifr&ref=ss_til&asins=4798118168" style="width:120px;height:240px;" scrolling="no" marginwidth="0" marginheight="0" frameborder="0"></iframe>
<iframe src="http://rcm-jp.amazon.co.jp/e/cm?lt1=_blank&bc1=000000&IS2=1&bg1=FFFFFF&fc1=000000&lc1=0000FF&t=takuojp02-22&o=9&p=8&l=as4&m=amazon&f=ifr&ref=ss_til&asins=4798121401" style="width:120px;height:240px;" scrolling="no" marginwidth="0" marginheight="0" frameborder="0"></iframe>

{% endraw %}
