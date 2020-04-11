---
toc: true
layout: post
tags: [Network, WireGuard]
author: proelbtn
title: WireGuardを利用した時のMTUについて
description: WireGuardを利用した時のMTUについて
---

2019年に登場した次世代のVPNとしてWireGuardというVPNがある。L3VPNしか張れないという制約はあるものの、Linuxの実装は4000行以下で済むほどシンプルなVPNである。そのシンプルさとは裏腹に、OpenVPNやIPsecと比べて高いスループットを実現している（[Whitepaper](https://www.wireguard.com/papers/wireguard.pdf)）。

WireGuard自体のセットアップは非常に簡単だが、MTUの計算周りで少し躓いた。あまりこの部分にたいしても言及されていないため、メモとして残しておく。


## encapsされた後のパケット構造

WireGuardはUDP上で動くVPNである。そのため、あるパケットがencapsされると次のようなパケットの構造になる。

![encaped-packet.png](/images/wireguard/encaped-packet.png)

元々のMTUが1500bytesだとすると、IPヘッダとUDPヘッダはそれぞれ20bytes, 8bytesなのでWireGuardのパケットに割ける領域は1472bytesになる。では、実際にWireGuardによって生成されるパケットの構造を見てみる。


## WireGuardのパケット構造

WireGuardでは、以下の4種類のパケットが生成されうる。可変長なパケットは一番下のtype4だけなのでそれについて詳しく見ていく。

![type1.png](/images/wireguard/type1.png)
![type2.png](/images/wireguard/type2.png)
![type3.png](/images/wireguard/type3.png)
![type4.png](/images/wireguard/type4.png)

type4の最後にある`packet`の部分が実際に暗号化されたL3パケットが入る。パケットの暗号化の流れは次のように行われる。初めに、暗号化したいパケットが16bytesの倍数になるように末尾に0をパディングする。その後、AEAD（認証付き暗号）で暗号化する。WireGuardでは、AEADの方式としてChaCha20Poly1305を利用しており、16bytesの認証用のタグが末尾につく。

![encrypt-procedure.png](/images/wireguard/encrypt-procedure.png)

ここまで分かれば適切なWireGuardのMTUが計算できる。前の節の例では、WireGuardのパケットに1472bytes割けることが分かっている。ここから`type`, `reserved`, `receiver`, `couter`の分で16bytes削られ、AEADで暗号化された暗号文に1456bytes割ける。暗号文には16bytesのタグが付いているので実際の暗号文には1440bytes使える。1440bytesは16bytesの倍数なので、WireGuardデバイスのパケットサイズとして1440bytesを指定すればパケットが問題なく転送される。

ここまでの計算をまとめると、WireGuardデバイスに指定すべきMTUを$y$, 元々のMTUを$x$とすると次のような関係が成り立つ。$\mathrm{floor}$は床関数（引数に与えられた小数以下の中で最も大きい整数）である。

\\[
    y = 16 \cdot \mathrm{floor}\left(\frac{x - 60}{16}\right)
\\]


## 実際の環境

僕の家はソフトバンク光を契約しており、IPv4の疎通性を提供するためにIPv4 over IPv6をしている。そのため、インターネットに出ていくIPv4のMTUは1500 - 40（IPv6ヘッダのパケット長）で1460bytesである。先程の関係式の$x$に1460を代入して計算を行うと、実際にWireGuardデバイスに対して指定すべきMTUが1392bytesだと計算が出来る。

## まとめ

WireGuardを利用してL3VPNを張る時のMTUの計算式やその計算式の根拠を示した。