---
title: "iptablesモードのkube-proxyはどのようにServiceリソースへのトラフィックをPodに転送するのか"
date: 2021-04-11T18:00:38+09:00
Description: ""
Tags: []
Categories: []
thumbnail: images/how-kube-proxy-in-iptables-mode-direct-traffic-to-pods-via-service-resources/network-namespaces-and-virtual-devices.png
DisableComments: false
---

&nbsp;


### はじめに

本記事は、[Dustin Specker](https://dustinspecker.com/about/)さんの[iptables: How Kubernetes Services Direct Traffic to Pods](https://dustinspecker.com/posts/iptables-how-kubernetes-services-direct-traffic-to-pods/)を一部改変し、学習用に書いたものであり、[CC BY-NC-SA 4.0](https://creativecommons.org/licenses/by-nc-sa/4.0/)の下に公開されています。

&nbsp;

本記事ではkubernetesのコンポーネントの一つであるkube-proxyがiptablesモードで稼働している際、どのようにiptablesを使用しているのか、また、どのようにアプリケーションサービスを提供しているPodに対してトラフィックを転送しているかを理解することを目的とします。

今回は、Serviceリソースのタイプの一つであるClusterIPを対象にします。

また、本記事のゴールはKubernetesで下記のようなServiceリソースを作成した際に、iptablesモードで稼働しているkube-proxyの動作をローカル環境上で再現することです。


```:service.yaml
apiVersion: v1
kind: Service
metadata:
  name: app-service
spec:
  clusterIP: 10.96.0.1
  selector:
    component: app
  ports:
    - protocol: TCP
      port: 8080
      targetPort: 8080
```

&nbsp;



### 検証環境

検証に使用した環境を以下に示します。

- Hardware: Raspberry Pi 4
- OS (e.g: cat /etc/os-release): Ubuntu 20.04.1 LTS (Focal Fossa)
- Kernel (e.g. uname -a): Linux 5.4.0-1023-raspi aarch64 GNU/Linux

&nbsp;



### 事前準備

物理デバイスから仮想デバイスへ、仮想デバイスから物理デバイスへ、また、仮想デバイス1から仮想デバイス2へなど異なるインターフェース間のパケット転送を行うことができるIPフォワーディングを有効にするために下記のコマンドを実行します。

```bash
sudo sysctl --write net.ipv4.ip_forward=1
```

&nbsp;



### 仮想デバイスを作成し、ネットワーク名前空間上でWebアプリケーションを実行する

まず、kubeletがCNIプラグインを利用して構築するPodネットワーク環境をローカル環境上に再現していきます。

KubernetesではPodごとに異なるネットワーク名前空間を作成しています。

本項では、手動でネットワーク名前空間を作成し、作成したネットワーク名前空間上で安易なWebアプリケーションサービスを実行することでKubernetesのPodを模倣してみます。

&nbsp;

先にGo言語で作成した安易なWebアプリケーションサービスを用意します。

Go言語の開発環境を構築します。

```bash
sudo apt install -y golang-go
```

&nbsp;

Go言語のWebフレームワークであるGinをインストールします。

```bash
sudo go get github.com/gin-gonic/gin
```

&nbsp;

下記のプログラムをmain.goとして保存します。

```:main.go
package main

import (
    "github.com/gin-gonic/gin"
    "net/http"
    "os/exec"
)

func main() {
    r:= gin.Default()
    out, _ := exec.Command("ip", "netns", "identify").Output()

    if len(out) == 1 {
        out = []byte("root\n")
    }

    r.GET("/", func(c *gin.Context) {
        c.JSON(http.StatusOK, gin.H{
        "message": "hello world " + "from " + string(out[:len(out)-1]) + " namespace",
        })
    })
    r.Run(":8080")
}
```

&nbsp;

動作確認としてWebアプリケーションサービスを実行します。

```bash
sudo go run main.go
```

&nbsp;

以下のような応答が返ってくればWebアプリケーションサービスは正常に動作しています。

```bash
$ curl -s localhost:8080 | jq
{
  "message":"hello world from root namespace"
}
```

&nbsp;

それでは、kubeletがCNIプラグインを利用して構築するPodネットワーク環境をローカル環境上に再現していきましょう。

Podネットワーク環境を再現する手順は下記の通りです。

1. 仮想ブリッジを作成する (仮想ブリッジ名： bridge\_eden)
2. ネットワーク名前空間を2つ作成する (ネットワーク空間名：netns\_adamとnetns\_eve)
3. 作成したネットワーク名前空間上でDNSの設定する
4. 仮想ブリッジ(bridge\_eden)に接続された2つのveth pairを作成する
5. IPアドレス 10.0.0.11をネットワーク名前空間 netns\_adam上のvethに割り当てる
6. IPアドレス 10.0.0.21をネットワーク名前空間 netns\_eve上のvethに割り当てる
7. ネットワーク名前空間上のデフォルトゲートウェイを設定する

上記の手順によって再現するPodネットワーク環境を下図に示しました。

![Network Namespaces and Virtual Devices](images/how-kube-proxy-in-iptables-mode-direct-traffic-to-pods-via-service-resources/network-namespaces-and-virtual-devices.png)

&nbsp;

#### 1. 仮想ブリッジを作成する

Linuxにはユーザが作成可能ないくつかの仮想デバイスがあり、その一つに物理イーサネットや仮想イーサネットなどのネットワークインタフェースがお互いに通信するための仮想ブリッジがあります。

再現する環境において、ネットワーク名前空間上で稼働するWebアプリケーションサービスが使用する仮想イーサネットが他のネットワークインタフェースと通信することができるように仮想ブリッジを作成します。

```bash
sudo ip link add dev bridge_eden type bridge
```

&nbsp;

次に、作成した仮想ブリッジに対してIPアドレスを割り当てます。

仮想ブリッジ bridege\_edenにIPアドレス 10.0.0.1/24を割り当てます。

```bash
sudo ip address add 10.0.0.1/24 dev bridge_eden
```

&nbsp;



#### 2. ネットワーク名前空間を2つ作成する

KubernetesのPodを模倣する上で必要な、Webアプリケーションサービスを実行するためのネットワーク名前空間を作成します。

2つのPodを作成するために、ネットワーク名前空間を2つ作成します。

```bash
sudo ip netns add netns_adam
sudo ip netns add netns_eve
```

&nbsp;

ネットワーク名前空間では、それぞれ自分自身のloopback(lo)デバイスを持っています。

loopbackデバイスは自分自身に対して送信したリクエストを自分自身で受信するための仮想デバイスです。

ネットワーク名前空間を作成した際、デフォルトでloopbackデバイスはリンクダウンしているため、リンクアップさせる必要があります。

```bash
sudo ip netns exec netns_adam ip link set dev lo up
sudo ip netns exec netns_eve  ip link set dev lo up
```
&nbsp;



### 3. 作成したネットワーク名前空間上でDNSの設定する

作成したネットワーク名前空間上でアドレス解決を行えるようにDNSの設定します。

```bash
sudo mkdir -p /etc/netns/netns_adam
sudo mkdir -p /etc/netns/netns_eve

echo "nameserver 8.8.8.8" | sudo tee -a /etc/netns/netns_adam/resolv.conf
echo "nameserver 8.8.8.8" | sudo tee -a /etc/netns/netns_eve/resolv.conf
```

&nbsp;



#### 4. 仮想ブリッジ bridge\_edenに接続された2つのvethを作成する

Linuxの仮想デバイスの一つに仮想イーサネットがあります。

仮想イーサネットは通常、ネットワーク名前空間の間を相互接続するために使用されます。

Podネットワーク環境の再現において、仮想イーサネットはルート名前空間と作成したあるネットワーク名前空間を相互接続することで、あるネットワーク名前空間上で稼働するWebアプリケーションサービスが他のネットワークインタフェースからのリクエストを処理できるようにします。

それでは、仮想イーサネットであるvethを作成していきます。

vethはペアで作成さるため、veth\_adamとveth\_ns\_adamが、veth\_eveとveth\_ns\_eveがペアとなった2つのvethを作成します。

```bash
sudo ip link add dev veth_adam type veth peer name veth_ns_adam
sudo ip link add dev veth_eve  type veth peer name veth_ns_eve
```

&nbsp;

作成したvethペアの片方である、veth\_adamとveth\_eveをルート名前空間上にある仮想ブリッジ bridge\_edenにリンクアップします。

```bash
sudo ip link set dev veth_adam master bridge_eden
sudo ip link set dev veth_eve  master bridge_eden

sudo ip link set dev veth_adam up
sudo ip link set dev veth_eve  up
```

&nbsp;

作成したvethペアのもう片方であるveth\_ns\_adamをネットワーク名前空間 netns\_adamに、veth\_ns\_eveをネットワーク名前空間 netns\_eveにリンクアップします。

```bash
sudo ip link set dev veth_ns_adam netns netns_adam
sudo ip link set dev veth_ns_eve  netns netns_eve

sudo ip netns exec netns_adam ip link set dev veth_ns_adam up
sudo ip netns exec netns_eve  ip link set dev veth_ns_eve  up
```

&nbsp;



#### 5. IPアドレス 10.0.0.11をネットワーク名前空間 netns\_adam上のvethに割り当てる

ネットワーク名前空間 netns\_adam上にある仮想イーサネットであるveth\_ns\_adamにIPアドレス 10.0.0.11/24を割り当てます。

```bash
sudo ip netns exec netns_adam ip address add 10.0.0.11/24 dev veth_ns_adam
```

&nbsp;



#### 6. IPアドレス 10.0.0.21をネットワーク名前空間 netns\_eve上のvethに割り当てる

ネットワーク名前空間 netns\_eve上にある仮想イーサネットであるveth\_ns\_eveにIPアドレス 10.0.0.21/24を割り当てます。

```bash
sudo ip netns exec netns_eve ip address add 10.0.0.21/24 dev veth_ns_eve
```

&nbsp;



#### 7. ネットワーク名前空間上のデフォルトゲートウェイを設定する

最後に、ルート名前空間上にある仮想ブリッジ bridge\_edenをリンクアップし、ネットワーク名前空間 netns\_adamとnetns\_eveに対してルート名前空間上にある仮想ブリッジ bridge\_edenのIPアドレスをデフォルトゲートウェイとして設定します。

```bash
sudo ip link set bridge_eden up
sudo ip netns exec netns_adam ip route add default via 10.0.0.1
sudo ip netns exec netns_eve  ip route add default via 10.0.0.1
```

&nbsp;



#### パケット送受信のためのiptablesのルール作成

作成した仮想ブリッジ bridge\_edenに対してパケットの送受信を許可するためのにiptablesのルールを作成します。

```bash
sudo iptables --table filter --append FORWARD --in-interface  bridge_eden --jump ACCEPT
sudo iptables --table filter --append FORWARD --out-interface bridge_eden --jump ACCEPT
```

&nbsp;

また、IPアドレス 10.0.0.0/24が送信元であるルート名前空間以外の他のネットワーク名前空間からのリクエストをマスカレードするiptablesのルールを作成します。

```bash
sudo iptables --table nat --append POSTROUTING --source 10.0.0.0/24 --jump MASQUERADE
```

&nbsp;



#### 動作確認

各ネットワーク名前空間上で稼働しているWebアプリケーションサービスがリクエスを受信し、レスポンスを送信することができるか確認します。

また、各ネットワーク名前空間から他のネットワーク名前空間上で稼働しているWebアプリケーションサービスにアクセスできるか確認します。


&nbsp;

ネットワーク名前空間 netns\_adam, netns\_eve、それぞれの空間上でWebアプリケーションサービスを起動します。

それぞれ別のターミナルを開いて実行して下さい。

```bash
sudo ip netns exec netns_adam go run main.go
```

```bash
sudo ip netns exec netns_eve  go run main.go
```

&nbsp;


ルート名前空間上からネットワーク名前空間 netns\_adm、netns\_eve、それぞれの空間上で稼働しているWebアプリケーションサービスにアクセスできる確認します。

```bash
$ curl -s 10.0.0.11:8080 | jq
{
  "message":"hello world from netns_adam namespace"
}

$ curl -s 10.0.0.21:8080 | jq
{
  "message":"hello world from netns_eve namespace"
}
```

&nbsp;

ネットワーク名前空間 netns\_adm上からネットワーク名前空間 netns\_eve上で稼働しているWebアプリケーションサービスにアクセスできるか確認します。

```bash
$ sudo ip netns exec netns_adam curl -s 10.0.0.21:8080 | jq
{
  "message":"hello world from netns_eve namespace"
}
```

&nbsp;

ネットワーク名前空間 netns\_eve上からネットワーク名前空間 netns\_adam上で稼働しているWebアプリケーションサービスにアクセスできるか確認します。

```bash
$ sudo ip netns exec netns_eve curl -s 10.0.0.11:8080 | jq
{
  "message":"hello world from netns_adam namespace"
}
```

&nbsp;

以上のような結果になればPodネットワーク環境が正常にローカル環境上に再現できています。

&nbsp;



### iptablesモードkube-proxyの動作を再現 ~iptablesに仮想IPアドレス追加する~

KubernetesにおいてServiceリソースが作成される際、ClusterIPはその作成された新しいServiceリソースに割り当てられます。

ClusterIPとは、概念的には仮想IPアドレスとのことです。

iptablesモードのkube-proxyは[仮想IPとサービスプロキシー](https://kubernetes.io/ja/docs/concepts/services-networking/service/#virtual-ips-and-service-proxies)で述べられているとおり、それらの仮想IPアドレスを取り扱うためのiptablesのルールを作成する責任があります。

それでは、iptablesモードのkube-proxyの動作を再現するためにiptablesのルールを作成していきます。

&nbsp;

ネットワークアドレス変換を行うNATテーブル上にServiceリーソスを再現するためのチェイン(ネットワーク経路)であるAPP-SERVICEをつくります。

```bash
sudo iptables \
  --table nat \
  --new APP-SERVICE
```

&nbsp;

作成したチェイン APP-SERVICEに対して、アプリケーションからリクエストを受け取れるようにチェインPREPUTINGとOUTPUTを追加します。

```bash
sudo iptables \
  --table nat \
  --append PREROUTING \
  --jump APP-SERVICE

sudo iptables \
  --table nat \
  --append OUTPUT \
  --jump APP-SERVICE
```

&nbsp;

次に、別のチェインであるEDEN-SVC-HTTPを作成します。
このチェインにネットワーク名前空間で稼働するWebアプリケーションサービスに対するチェインを紐付けていきます。

```bash
sudo iptables \
  --table nat \
  --new EDEN-SVC-HTTP
```

&nbsp;

先程作成したチェイン APP-SERVICEに対してチェイン EDEN-SVC-HTTPを紐付けます。

**--destination**で指定している仮想IPアドレス **10.96.0.1**がClusterIPとなります。

```bash
sudo iptables \
  --table nat \
  --append APP-SERVICE \
  --destination 10.96.0.1 \
  --protocol tcp \
  --match tcp \
  --dport 8080 \
  --jump EDEN-SVC-HTTP
```

&nbsp;

ネットワーク名前空間 netns\_adamで稼働するWebアプリケーションサービスに対して、ネットワーク経路として振る舞うためのチェイン ADAM-HTTPを作成し、チェイン EDEN-SVC-HTTPに紐付けます。

```bash
sudo iptables \
  --table nat \
  --new ADAM-HTTP

sudo iptables \
  --table nat \
  --append ADAM-HTTP \
  --protocol tcp \
  --match tcp \
  --jump DNAT \
  --to-destination 10.0.0.11:8080

sudo iptables \
  --table nat \
  --append EDEN-SVC-HTTP \
  --jump ADAM-HTTP
```

&nbsp;

最後に、ネットワーク名前空間 netns\_eveで稼働するWebアプリケーションサービスに対して、ネットワーク経路として振る舞うためのチェイン EVE-HTTPを作成し、チェイン EDEN-SVC-HTTPに紐付けます。

チェイン EVE-HTTPをチェイン EDEN-SVC-HTTPに紐付ける際に、ルールの一番最初に挿入していることに注意して下さい。(**--insert EDEN-SVC-HTTP 1**の部分)　

iptablesのルールは上から順番通りに適用されます。

最初にこのルールを設定することで、このチェイン EVE-HTTPのルールが適用される確率が50%になります。

したがって、ClusterIPとして振る舞っている仮想IPアドレス **10.96.0.1**へのアクセスは、Podとして稼働しているWebアプリケーションサービスに対するトラフィックをランダムに分散させることができるようになります。

```bash
sudo iptables \
  --table nat \
  --new EVE-HTTP

sudo iptables \
  --table nat \
  --append EVE-HTTP \
  --protocol tcp \
  --match tcp \
  --jump DNAT \
  --to-destination 10.0.0.21:8080

sudo iptables \
  --table nat \
  --insert EDEN-SVC-HTTP 1 \
  --match statistic \
  --mode random \
  --probability 0.5 \
  --jump EVE-HTTP
```

&nbsp;



### 動作確認

これまで、kubeletがCNIプラグインを利用して構築するPodネットワーク環境をローカル環境上に再現し、また、KubernetesがServiceリソースを作成した際に、iptablesモードで稼働しているkube-proxyの動作をローカル環境上で再現してきました。

最後に、ClusterIPと見立てて設定した仮想IPアドレス **10.96.0.1:8080**にアクセスしてトラフィックが分散していることを確認し、iptablesを介した負荷分散が行われていることに感動して下さい！

```bash
ubuntu@ubuntu:~$ curl -s 10.96.0.1:8080 | jq
{
  "message": "hello world from netns_adam namespace"
}
ubuntu@ubuntu:~$ curl -s 10.96.0.1:8080 | jq
{
  "message": "hello world from netns_eve namespace"
}
ubuntu@ubuntu:~$ curl -s 10.96.0.1:8080 | jq
{
  "message": "hello world from netns_eve namespace"
}
ubuntu@ubuntu:~$ curl -s 10.96.0.1:8080 | jq
{
  "message": "hello world from netns_adam namespace"
}
ubuntu@ubuntu:~$ curl -s 10.96.0.1:8080 | jq
{
  "message": "hello world from netns_eve namespace"
}
ubuntu@ubuntu:~$ curl -s 10.96.0.1:8080 | jq
{
  "message": "hello world from netns_adam namespace"
}
```

&nbsp;



### 後片付け
#### iptablesのチェインとルールの削除

本検証で作成したiptablesのチェインやルールを削除します。

まず、チェイン上のルールを削除します。
下記のコマンドを実行し、チェイン上のルールにつけられた番号を確認して下さい。

```bash
sudo iptables -t nat -L --line-numbers
```

&nbsp;

次に、チェイン上のルールを削除します。
下記のコマンドに沿ってルールを削除して下さい。

```bash
sudo iptables --table nat --delete <chain_name> <rule_number>
```

&nbsp;

最後に、チェインを削除します。
ルールが紐付いているチェインは削除できませんのであらかじめチェインに紐付いているすべてのルールの削除して下さい。

```bash
sudo iptables --table nat --delete-chain <chain_name>
```

&nbsp;



#### 仮想デバイスとネットワーク名前空間の削除

次に、Podネットワーク環境を構築する際に作成した仮想イーサネット veth、ネットワーク名前空間と仮想ブリッジ bridge\_edenを削除します。


```bash
sudo ip link del dev veth_ns_adam
sudo ip link del dev veth_ns_eve

sudo ip netns del netns_adam
sudo ip netns del netns_eve
sudo rm -rf /etc/netns

sudo ip link del dev bridge_eden
```

&nbsp;



#### IPフォワーディングの無効化
最後に、IPフォワーディングを無効化します。

```bash
sudo sysctl --write net.ipv4.ip_forward=0
```


&nbsp;



### まとめ

本記事ではkubernetesのコンポーネントの一つであるkube-proxyがiptablesモードで稼働している際、どのようにiptablesを使用しているのか、また、Serviceリソースのタイプの一つであるClusterIPがどのようにアプリケーションサービスを提供しているPodに対してトラフィックを転送しているかを理解することを目的としました。

また、kubeletがCNIプラグインを利用して構築するPodネットワーク環境をローカル環境上で再現し、iptablesコマンドを使用してKubernetesでServiceリソースを作成した際に、iptablesモードで稼働しているkube-proxyの動作をローカル環境上で再現しました。

最後に、ClusterIPと見立てて設定した仮想IPアドレスに対するアクセスのトラフィックが分散されていることを確認し、iptablesを介した負荷分散を行うことができました。

&nbsp;

さて、次の疑問は、IPVSモードのkube-proxyはどのようにServiceリソースを経由してトラフィックをPodに転送するのでしょうか？

&nbsp;



### Copyright
本記事は、[Dustin Specker](https://dustinspecker.com/about/)さんの[iptables: How Kubernetes Services Direct Traffic to Pods](https://dustinspecker.com/posts/iptables-how-kubernetes-services-direct-traffic-to-pods/)を一部改変し、学習用に書いたものであり、[CC BY-NC-SA 4.0](https://creativecommons.org/licenses/by-nc-sa/4.0/)の下に公開されています。

&nbsp;
