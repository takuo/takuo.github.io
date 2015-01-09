---
layout: post
title: "ドメイン名が変わりました"
date: 2015-01-09 20:52
comments: true
categories: [ network ]
---
ドメインを変更することになりました。  
新しいドメインは n6e.be です。

<!-- more -->

ga.vg ドメインは2文字でそんなに高額でもなく気に入っていたんですが、昨年のいつ頃だったかに .vg ドメインの管理団体が破産したとか何とかで変更になり、
以後の更新料は年間1万円を超えるようになってしまいました。  
さすがに年1万はな〜ってなって変更することにいたしました。  

今度も3文字で比較的短いドメインなのですが、やっぱり2文字がよかったなという思いは残ります。  

さて、今回ドメイン取得にあたり、AmazonのRoute53サービスを利用してみました。  
ちなみにRoute53はバックエンドレジストラに[Gandi](http://www.gandi.net)を利用しているようです。 

.be ドメインは $9 で取得でき、他のドメイン販売業者と比べても対して差がありません。  
.jp ドメインも $90 で大きく変わらないのですが、まぁ円安ということもあり、円換算すると少々高くなりますね。  

取得自体はAWSのマネジメントコンソールから適当にポチポチやってれば買えます。  
新規ドメインの場合は、自動的にDNSのホスティングもRoute53になります。  
もちろん外のDNSに変更も可能ですが、APIもあるしDDNSもやりやすいだろうと思ってこのまま運用することにしました。  
費用としては $0.5/zoneに加えて10億クエリまでは $0.4 で、だいたい $1/month で運用できていますし、気になる額でもありません。  
{% img /assets/screenshot/aws-route53-price.png %}

DDNSなどに用いるコマンドラインツールは [cli53](https://github.com/barnybug/cli53) というPythonツールを使っています。  
``` bash command line
sudo pip install cli53
```
という具合にpipでインストール。  
あとは /etc/ppp/ip-up.d/ddns あたりに適当にかいとけばDDNSできます。  

``` bash /etc/ppp/ip-up.d/ddns
#!/bin/sh
set -e

# dynamic.n6e.be をTTL 90で登録
cli53 rrcreate n6e.be dynamic A $PPP_LOCAL --ttl 90 --replace
```
便利です。


今後共よろしくお願いいたします。  
