---
title: "本ブログを運用している自宅インフラ『MAKINA』の紹介"
date: 2020-05-16T20:41:40+09:00
Description: ""
Tags: []
Categories: []
thumbnail: /images/introducing-the-architecture-of-raspberry-pi-kubernetes-cluster-named-makina-that-operates-this-blog/architecture.png
DisableComments: false
---

&nbsp;

本稿は、このブログを運用している自宅インフラのアーキテクチャーの紹介をしたいと思います。

すでに[このブログの最初の記事](http://techblog.sosomasox.com/posts/starting-a-tech-blog/)で言及しておりますが、本ブログはRaspberry Piで構築した高可用性Kubernetesクラスターをインフラとして運用してます。

したがって、本稿ではRaspberry Piで構築したKubernetesクラスターのアーキテクチャーとネットワーク構成、このブログを公開するために構築したシステムの構成について紹介します。

ちなみにこのインフラを『MAKINA』と呼んでいます。名前に意味はありません。

&nbsp;



### アーキテクチャーについて

本インフラのアーキテクチャーは大きく **3つのコンポーネント** で構成されています。

Kubernetesクラスターを構成要素であるアクティブ/アクティブ構成で冗長化された**Kubernetesマスターノード群**と**Kubernetesワーカーノード群**、これらで構成されたKubernetesクラスターのkube-apiserverに対するAPIリクエストを複数のマスターノードに分散させるためのアクティブ/スタンバイ構成で冗長化された**ロードバランサ**です。

&nbsp;

![Architecture](/images/introducing-the-architecture-of-raspberry-pi-kubernetes-cluster-named-makina-that-operates-this-blog/architecture.png)

&nbsp;



### Kubernetesワーカーノード群に関して

Kubernetesワーカーノード群はKubernetesマスターノード群によってスケジューリングされたPodを動作させます。

Kubernetesマスターノード上ではPodを動作させず、全てのPodはワーカーノード上で動作するようにしています。

このブログも一連のPod群として管理され、ワーカーノード上で動作しています。

&nbsp;



### Kubernetesマスターノード群に関して

Kubernetesのマスターノードはクラスターに対するの操作機能の提供(kube-apiserver)、クラスターに関する情報の管理(etcd)、PodのスケジューリングやPodを適切なワーカーノードに再割り当てるための管理(kube-scheduler)、ワーカーノードの状態監視(kube-controller-manager)といったKubernetesクラスター全体の管理を行っています。

シングルコントロールプレーンクラスターでは1台のマスターノードが故障した場合、クラスターの管理を継続させることができません。
システムを構成する部分のうち、そこに障害が発生するとシステム全体が停止してしまう部分を **単一障害点 (SPOF : Single Point of Failure)** と呼びますが、シングルコントロールプレーンクラスターではこの1台のマスターノードが単一障害点となります。

そのため本インフラでは、コントロールプレーンを複数用意することでマスターノードに対して冗長化構成をとり、あるマスターノードが故障した場合でも他のマスターノードがクラスターの管理を継続することを可能にすることで単一障害点をなくし、高可用性を実現させています。

&nbsp;

![Redundant Kubernetes Master Nodes](/images/introducing-the-architecture-of-raspberry-pi-kubernetes-cluster-named-makina-that-operates-this-blog/redundant_kubernetes_master_nodes.png)

&nbsp;

本アーキテクチャーのKubernetesマスターノード群は**アクティブ/アクティブ構成**を採ることで冗長化させます。
アクティブ/アクティブ構成とは、同じ機能を持つシステムコンポーネントを複数用意し、これらのコンポーネントを常に同時に稼動させるようなシステム構成をとる冗長化手法の一つです。
このような冗長化手法を採ることで各コンポーネントに対する負荷分散がもたらすシステムパフォーマンスの向上や、あるコンポーネントが機能を継続することができなくなった場合でも他の同じ機能を持つコンポーネントが代わりに機能を提供することができるため、システムの可用性を高めることができます。

しかし、Kubernetesマスターノード群に対してアクティブ/アクティブ構成をとる場合には以下のの項目に対処する必要があります。

1. **クラスターに関する情報をマスターノード間で共有する必要がある**
2. **クラスターの操作に関するAPIリクエストを各マスターノードのkube-apiserverに振り分ける必要がある**

本アーキテクチャーでは上記の項目に対する対処方法として、以下の方法を採っています。

**1.に関して、マスターノード同士がetcdクラスター上に情報を保存することでKubernetesクラスターに関する情報を共有させます。**
etcdクラスターを構築する方法として、[Kubernetes公式ドキュメントに記載されている方法](https://kubernetes.io/ja/docs/setup/production-environment/tools/kubeadm/high-availability/)があります。
本Kubernetesクラスターの構築にあたっては、公式ドキュメントに記載されているetcdクラスターを構築する方法からetcdのメンバーとコントロールプレーンを結合した構成である積み重なったコントロールプレーンを使う方法を採用しています。

**2.に関しては、負荷分散によってAPIリクエストの振り分けを行うことができるためロードバランサを導入することで対処することが可能です。**

しかし、本インフラの高可用性を実現させるためにはこの導入するロードバランサの構築に対していくつかの制約があります。

- **マスターノードが正常に稼働しているか確認することができる監視機能がなければならない**
- **障害が発生したマスターノードに対しては処理を振り分けないようにしなければならない**
- **ロードバランサ自体が単一障害点とならないように冗長化しなければならない**

本アーキテクチャーでは、上記の制約を満たすために**アクティブ/スタンバイ構成で冗長化されたロードバランサ**を導入することで対処しています。

&nbsp;



## ロードバランサに関して

本インフラで稼働しているロードバランサは、アクティブ/アクティブ冗長化構成Kubernetesマスターノード群上で動作するkube-apiserverに対するAPIリクエストを受け取り、各マスターノードのkube-apiserverにリクエストを振り分ける役割を担っています。

本アーキテクチャーにおいて、高可用性を実現させるための制約を満たすために負荷分散機能やHTTPリバースプロキシとしての機能、サーバに対するの監視機能、ヘルスチェック機能による非稼働中のサーバに対するリクエスト振り分けの停止などを提供するオープンソースのTCP/HTTPプロキシソフトウェアである[**HAProxy**](http://www.haproxy.org/)を利用することでロードバランサを構築しています。

また、本アーキテクチャーにおけるロードバランサの構築に対して、 **"ロードバランサ自体が単一障害点とならないように冗長化しなければならない"** という制約がありました。
冗長化手法に関しては、[アクティブ/アクティブ冗長化構成Kubernetesマスターノード群](#アクティブアクティブ冗長化構成kubernetesマスターノード群)にで **アクティブ/アクティブ構成** という手法を説明しました。
これに対して、導入するロードバランサでは **アクティブ/スタンバイ構成** という冗長化手法を採っています。
アクティブ/スタンバイ構成とは、同じ機能を持つシステムコンポーネントを複数用意し、ある一つのコンポーネントだけを稼働状態とし、他のコンポーネントは待機状態にしておくようなシステム構成をとる冗長化手法の一つです。
このような冗長化手法は、何らかの原因によって稼動状態のシステムコンポーネントに障害が発生したとき、待機状態のシステムコンポーネントを稼動状態へと切り替えることによってシステムの稼働を継続させることができるため、システムの可用性を高めることができます。

本アーキテクチャーでは、仮想IPアドレス(VIP)の引き継ぎ機能やVRRPによるサーバの死活監視機能、サービスのヘルスチェック機能などを提供するソフトウェアである[**Keepalived**](https://www.keepalived.org/)を利用することで **アクティブ/スタンバイ冗長化構成ロードバランサ** を構築しています。

&nbsp;

![Redundant Load Balancers](/images/introducing-the-architecture-of-raspberry-pi-kubernetes-cluster-named-makina-that-operates-this-blog/redundant_load_balancers.png)

&nbsp;



### ネットワーク構成について

本インフラのネットワーク構成を下図に示します。

&nbsp;

![Network Architecture](/images/introducing-the-architecture-of-raspberry-pi-kubernetes-cluster-named-makina-that-operates-this-blog/network_architecture.jpg)

&nbsp;

自宅のネットワーク環境の外にラズパイKubernetesクラスター用のネットワーク環境を構築しています。
そのため、ラズパイKubernetesクラスター用ネットワークを自宅ルータに接続するためにゲートウェイデバイスを設置しています。
このゲートウェイデバイスにもRaspberry Piを使用しています。
このゲートウェイとしてのラズパイでには標準で搭載されているEthernetポートに加えて、[USB3.0対応の有線LANアダプター](https://www.amazon.co.jp/gp/product/B0784RHQDL/ref=ppx_yo_dt_b_asin_title_o06_s00?ie=UTF8&psc=1)を搭載しています。

現状では1台のゲートウェイで運用していますが、今後、冗長化構成を採ることを計画しています。

デフォルトの設定では、一方のEthernetポート(eth0)は自宅ルータと同様のネットワーク環境下にあるのでゲートウェイデバイスとして稼働しているラズパイで自体はインターネットへの接続が可能ですが、もう一方のEthernetポート(eht1)の下で構成されているラズパイKubernetesクラスター用ネットワーク環境下にあるラズパイらは自宅ルータと同様のネットワークではないため、インターネットに接続することができません。

そのため、ゲートウェイデバイスとして稼働するラズパイでiptablesに関する下記の設定を行っています。

```bash
$ sudo iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE
$ sudo iptables -A FORWARD -i eth1 -o eth0 -m state --state RELATED,ESTABLISHED -j ACCEPT
$ sudo iptables -A FORWARD -i eth0 -o eth1 -j ACCEPT
```

&nbsp;



### 本ブログを公開しているシステムの構成について

本ブログでは、独自ドメインでサービスを公開するためにAmazon Route 53を利用しています。
また、本ブログにアクセスためのエンドポイントとしてAmazon EC2を利用しており、1台のインスタンスをHTTPリバースプロキシとして稼働させています。
Amazon EC2では一番料金が安いインスタンスを使用しています。

一方で、ラズパイKubernetesクラスター上で稼働している本ブログを外部に公開する手段として[**ngrok**](https://ngrok.com/)というサービスを利用しています。
ngrokはローカル環境で稼働しているサーバを外部公開することができるサービスです。

HTTPリバースプロキシの転送先をngrokで取得したURLに設定することで、独自ドメインで本ブログにアクセスできるようになっています。

&nbsp;

![Blog System Architecture](/images/introducing-the-architecture-of-raspberry-pi-kubernetes-cluster-named-makina-that-operates-this-blog/blog_system_architecture.jpg)

&nbsp;



### 最後に
本稿では、このブログを公開するために構築したシステムの構成について紹介しました。

次回はこのブログを公開するためにKubernetes上で稼働しているサービスのアーキテクチャーを紹介したいと思います。

&nbsp;
