---
layout: post
title: "さくらのVPS x LXC x IPv6"
date: 2015-04-08 15:39
comments: true
categories: [ Network ]
---
こんにちは。  
さくらのVPSで使えていた[6rdトライアルが3/31で終わってしまいました](http://research.sakura.ad.jp/6rd-trial/)。  

ところで、だいぶ前からさくらのVPSではネイティブなIPv6アドレスが配布され、利用できていました。([「さくらのVPS」大阪リージョン IPv6アドレス提供開始のお知らせ](http://www.sakura.ad.jp/news/sakurainfo/newsentry.php?id=925))  
しかしながら、配布されているアドレスは1個のみで使い勝手がよくなく、それが6rdを最後まで使っていた理由でもありました。  

まぁサービスが終わってしまったのなら移行せざるを得ないということで重い腰を上げて作業した次第です。

<!-- more -->

## さくらVPSのIPv6

前述の通り、現在さくらVPSで使えるIPv6はネイティブ接続になっています。  
単純にVPSでIPv6アドレスを使うだけなら以下の設定だけで十分です。

``` bash /etc/network/interfaces
iface eth0 inet6 static
        address 2403:3a00:202:1202:xxx:xxx:xxx:xxx
        netmask 64
        gateway fe80::1
```

ですが、うちのようにVPSでLXCを構成して、各種サービスはそれぞれのコンテナ内でうごかしているような環境の場合、IPv6でサービスにアクセスするには各コンテナにアドレスを振るか、IPv4のようにNATする必要があります。

6rdトライアルでは/64のネットワークアドレスを使えたのでブリッジに対してアドレスを広告してあげれば済みましたが、現状のIPv6は1個しかアドレスがないのでこれをどうにかしてshareするしかありません。  
(BSD系のjailではうまくshareできるようですがよく知りません)

## IPv6でNAT
 
IPv6でNATなんていうのは極めてnonsenseで非常にevilですがアドレスが一個しかない以上仕方がありません。  
さくらの人間は[RFC6177](https://tools.ietf.org/html/rfc6177)を読んでいるのでしょうか。  
ともかく、[Linux(3.9.0以降, iptables-1.4.18以降)ではIPv6でもNATできる](http://mirrors.bieringer.de/Linux+IPv6-HOWTO/nat-netfilter6..html)ようなのでやってみることにしました。  

とりあえずカーネルとiptablesのバージョンに問題なければIPv4と全く同じようにNATできるとのことです。

まずLXCのコンテナと通信するためのbridgeを確認します。

    % brctl show lxcbr0
    bridge name	bridge id		STP enabled	interfaces
    lxcbr0		8000.fe59f8967f7c	no		vm00eth0
         							vm01eth0
         							vm02eth0
         							vm03eth0
         							vm04eth0
    % ifconfig lxcbr0
    lxcbr0    Link encap:イーサネット  ハードウェアアドレス fe:59:f8:96:7f:7c 
              inetアドレス:10.0.0.1 ブロードキャスト:10.0.0.255  マスク:255.255.255.0
              inet6アドレス: fe80::fc59:f8ff:fe96:7f7c/64 範囲:リンク

vm00内の`eth0`は…

    % ifconfig eth0
    eth0      Link encap:イーサネット  ハードウェアアドレス 00:ff:00:00:00:03 
              inetアドレス:10.0.0.3 ブロードキャスト:10.0.0.255  マスク:255.255.255.0
              inet6アドレス: fe80::2ff:ff:fe00:3/64 範囲:リンク

まぁこんな感じですので、`lxcbr0` のリンクローカルを使って `fe80::2ff:ff:fe00:3` へDNATしたいと思います。  

    # ip6tables -t nat -I PREROUTING -p tcp -d 2403:3a00:202:1202:xxx:xxx:xxx:xxx -j DNAT --to-destination fe80::2ff:ff:fe00:3

とやれば良さそうに思いましたが、うまく行きませんでした。    
*tcpdump*で覗けばわかりますが、*neighbor solicitation*に失敗しています。  
はい、リンクローカルはデバイス(*scope id*)を指定しなければうまく通信できません。  
*ping6* でもリンクローカルは `-I eth0` でインターフェースを指定したり、アドレスに`%eth0`をつけるなどしないと通りませんね。  
そんなわけで、  

    # ip6tables -t nat -I PREROUTING -p tcp -d 2403:3a00:202:1202:xxx:xxx:xxx:xxx -j DNAT --to-destination fe80::2ff:ff:fe00:3%lxcbr0
    ip6tables v1.4.21: Bad IP address "fe80::2ff:ff:fe00:3%lxcbr0"

などとやってみますが、拒否されました。  
なるほど、iptablesでは*scope id*を指定できないのですね。  
というわけで、リンクローカルはあきらめて[Unique Local IPv6 Unicast Address (ULA)](https://www.nic.ad.jp/ja/newsletter/No30/022.html)を使います。  
ともかく使うアドレスは`fd00::/8`でよさそうです。  

コンテナからの戻りのrouteも必要ですので*masquerade*します。

    # ip addr add fd00::1/8 dev lxcbr0
    # ip6tables -t nat -I PREROUTING -p tcp -d 2403:3a00:202:1202:xxx:xxx:xxx:xxx --dport 80 -j DNAT --to-destination fd00::3
    # ip6tables -t nat -I PREROUTING -p udp -d 2403:3a00:202:1202:xxx:xxx:xxx:xxx --dport 53 -j DNAT --to-destination fd00::3
    # ip6tables -t nat -I POSTROUTING -s fd00::/8 -o eth0 -j MASQUERADE

LXCコンテナでは

    # ip addr add fd00::3/8 dev eth0
    # ip -6 ro add default via fd00::1 dev eth0

結局の所、ULAをプライベートアドレスに読みかえれば、IPv4と同じオペレーションでNATができるようになるということです。

## フィルタリングファイウォール

IPv6でアクセスできることを確認したらIPv4と同じようにフィルタリングしておきましょう。  
これもIPv4のiptablesとまったく同じように設定します。 

    # ip6tables -I INPUT -P DROP   # default でDROP
    # ip6tables -I FORWARD -P DROP # default でDROP
    # ip6tables -A FORWARD -p tcp -m multiport --dports 80,443 -j ACCEPT
    # ip6tables -A FORWARD -p udp -m multiport --dports 53,123 -j ACCEPT
    (以下略)

以上です。  
あとはさくらVPSがせめて/64くれるようになることを祈るだけですね。

