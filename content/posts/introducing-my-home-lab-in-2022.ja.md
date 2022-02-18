---
title: "自宅ラボ環境の紹介 in 2022"
date: 2022-02-18T00:00:00+09:00
Description: ""
Tags: []
Categories: []
thumbnail: /images/introducing-my-home-lab-in-2022/thumbnail.png
DisableComments: false
---

&nbsp;

# はじめに

構成や　変わったので



&nbsp;



# アーキテクチャー

雑にまとめるとこんなかんじ。

&nbsp;

![Architecutre](/images/introducing-my-home-lab-in-2022/architecture.png)

&nbsp;



## EdgerRouter X

こいつが自宅ラボのインターネット接続を担っている。
自宅ネットワークを用途別に分けたり、本ブログを外部に公開する設定を行っている。

#### DNSフォワーディング

LANインタフェースからのDNSクエリは、後述するキャッシュDNSサーバにフォワードするように設定されている。


#### ポートフォワーディング

WANインターフェスpppoe0に来たHTTP/HTTPS宛のパケットは、後述するラズパイKubernetes上で稼働しているNGINX Ingress Controllerにフォワードするように設定されている。

本ブログは、Kubernetes上で運用しているのでこの設定でブログを外部公開している。

&nbsp;



## DNS

自宅ラボ環境では4台のDNSサーバが稼働している。
自宅で稼働しているサーバ機器やサービスに付与した独自ドメインの名前解決を行う。


#### ns1.home.arpa

ゾーン *"home.arpa."* を管理するプライマリ権威DNSサーバ。
サーバ機器や自宅で運用するサービスのドメインを管理している。


#### ns2.home.arpa

ゾーン "*home.arpa.*" を管理するセカンダリ権威DNSサーバ。
ゾーン転送によって ns1.home.arpa からゾーン情報を取得。


#### dns.k8s.home.arpa

ゾーン *"hoem.arpa."* を管理する権威DNSサーバによって委任された、
ゾーン *"k8s.home.arpa."* を管理する権威DNSサーバ。

ラズパイKubernetes用のサーバにつけるドメインを管理している。


#### ns-cache.home.arpa

独自ドメインのキャッシュDNSサーバ。

自宅で稼働している全ての機器に対して名前解決を行う。

&nbsp;


## リバースプロキシサーバ

自宅環境で稼働しているいろいろなサービスのHTTP接続用のリバースプロキシサーバ。

後述するMinIOやGrafana、KibanaなどWeb UIが備わっているサービスに対してWebブラウザからポート指定なしで(80番ポートで)アクセスできるようにプロキシしている。

&nbsp;


## MinIO (オブジェクトストレージ)

後述するLonghornやVeleroのバックアップストアとして稼働。
分散構成はとっていない。

![MinIO Dashboard](/images/introducing-my-home-lab-in-2022/minio_dashboard.png)

&nbsp;



## メトリクス監視

EdgeRouterやDNSサーバ、ラズパイKubernetes用のさーばなど、本環境で稼働しているサーバのメトリクス監視を担う。

Prometheus + 各種Exporter で構築。

![Prometheus Targets](/images/introducing-my-home-lab-in-2022/prometheus_targets.png)

ダッシュボードにはGrafanaを利用。

![Grafana Dashboard](/images/introducing-my-home-lab-in-2022/grafana_dashboard.png)

&nbsp;

AlertmanagerやPushgatewayも構築済み。
(アラートルールちゃんと整備してない)

![Alertmanager Alert](/images/introducing-my-home-lab-in-2022/alertmanager_alert.png)

&nbsp;



## ログ収集

Kubernetesで　コンテナのログ


![Kibana Dashborad](/images/introducing-my-home-lab-in-2022/kibana_dashboard.png)

&nbsp;



## Wi-Fiルータ

Wi-Fiルータは [tp-link Wi-Fiルータ Archer AX73](https://www.tp-link.com/jp/home-networking/wifi-router/archer-ax73/) をブリッジモードで動作させ利用している。

スマホやノートPCなど無線機器が接続されてる。


&nbsp;



## ラズパイ Kubernetes


USB接続のSSDを使用したRaspberry Pi 4 Model B 8GB に The Hard Way で構築。

コントロールプレーン3台とノード5台で構成。

Kubernetes のデータストアである etcd は、コントロールプレーンに併設し、etcd クラスタを構成。

また、kube-apiserver 用のロードバランサはHAPorxy + Keepalived の冗長構成でコントロールプレーンに併設。


```bash
$ kubectl get nodes -o wide
NAME                            STATUS   ROLES                     AGE     VERSION   INTERNAL-IP      EXTERNAL-IP   OS-IMAGE             KERNEL-VERSION     CONTAINER-RUNTIME
control-plane-1.k8s.home.arpa   Ready    control-plane,etcd,node   7d11h   v1.23.1   192.168.114.11   <none>        Ubuntu 20.04.3 LTS   5.4.0-1050-raspi   containerd://1.5.5
control-plane-2.k8s.home.arpa   Ready    control-plane,etcd,node   7d11h   v1.23.1   192.168.114.12   <none>        Ubuntu 20.04.3 LTS   5.4.0-1050-raspi   containerd://1.5.5
control-plane-3.k8s.home.arpa   Ready    control-plane,etcd,node   7d11h   v1.23.1   192.168.114.13   <none>        Ubuntu 20.04.3 LTS   5.4.0-1050-raspi   containerd://1.5.5
node-1.k8s.home.arpa            Ready    node                      7d11h   v1.23.1   192.168.114.14   <none>        Ubuntu 20.04.3 LTS   5.4.0-1050-raspi   containerd://1.5.5
node-2.k8s.home.arpa            Ready    node                      7d11h   v1.23.1   192.168.114.15   <none>        Ubuntu 20.04.3 LTS   5.4.0-1050-raspi   containerd://1.5.5
node-3.k8s.home.arpa            Ready    node                      7d11h   v1.23.1   192.168.114.16   <none>        Ubuntu 20.04.3 LTS   5.4.0-1050-raspi   containerd://1.5.5
node-4.k8s.home.arpa            Ready    node                      7d11h   v1.23.1   192.168.114.17   <none>        Ubuntu 20.04.3 LTS   5.4.0-1050-raspi   containerd://1.5.5
node-5.k8s.home.arpa            Ready    node                      7d11h   v1.23.1   192.168.114.18   <none>        Ubuntu 20.04.3 LTS   5.4.0-1050-raspi   containerd://1.5.5
```

&nbsp;


## MetalLB

ベアメタル環境の Kubernetes において、 
Service リソースで type: LoadBalaner を使用するために導入。

これにより、Service リソースにIPアドレスが付与され、
Kubernetes クラスタ外からでも Kubernetes上で稼働しているサービスにアクセスすることができる。

L2モードとBGPモードがサポートされているが、
EdgeRouter X がBGPに対応しているのでBGPモードで稼働。


```vim
apiVersion: v1
kind: ConfigMap
metadata:
  namespace: metallb-system
  name: config
data:
  config: |
    peers:
    - my-asn: 65002
      peer-asn: 65001
      peer-address: 192.168.114.254
    address-pools:
    - name: my-ip-space
      protocol: bgp
      addresses:
      - 192.168.115.1-192.168.115.254
```

&nbsp;

```bash
ubnt@edgerouter-x:~$ show ip bgp summary
BGP router identifier 192.168.114.254, local AS number 65001
BGP table version is 14
2 BGP AS-PATH entries
0 BGP community entries
32  Configured ebgp ECMP multipath: Currently set at 32
1  Configured ibgp ECMP multipath: Currently set at 1

Neighbor                 V   AS   MsgRcv    MsgSen TblVer   InQ   OutQ    Up/Down   State/PfxRcd
192.168.114.11           4 65002  123        114      14      0      0  00:55:28              11
192.168.114.12           4 65002  123        116      14      0      0  00:55:27              11
192.168.114.13           4 65002  123        116      14      0      0  00:55:25              11
192.168.114.14           4 65002  123        115      14      0      0  00:55:19              11
192.168.114.15           4 65002  123        115      14      0      0  00:55:26              11
192.168.114.16           4 65002  123        116      14      0      0  00:55:27              11
192.168.114.17           4 65002  123        116      14      0      0  00:55:27              11
192.168.114.18           4 65002  123        115      14      0      0  00:55:26              11

Total number of neighbors 8

Total number of Established sessions 8
```

&nbsp;

```bash
ubnt@edgerouter-x:~$ show ip route
Codes: K - kernel, C - connected, S - static, R - RIP, B - BGP
       O - OSPF, IA - OSPF inter area
       N1 - OSPF NSSA external type 1, N2 - OSPF NSSA external type 2
       E1 - OSPF external type 1, E2 - OSPF external type 2
       > - selected route, * - FIB route, p - stale info

IP Route Table for VRF "default"
K    *> 0.0.0.0/0 [0/0] via pppoe0
C    *> xxx.xxx.xxx.xxx/32 is directly connected, pppoe0
C    *> 127.0.0.0/8 is directly connected, lo
C    *> 192.168.110.0/24 is directly connected, eth0
C    *> 192.168.111.0/24 is directly connected, eth1
C    *> 192.168.112.0/24 is directly connected, eth2
C    *> 192.168.113.0/24 is directly connected, eth3
C    *> 192.168.114.0/24 is directly connected, eth4
B    *> 192.168.115.1/32 [20/0] via 192.168.114.18, eth4, 00:56:06
     *>                  [20/0] via 192.168.114.17, eth4, 00:56:06
     *>                  [20/0] via 192.168.114.16, eth4, 00:56:06
     *>                  [20/0] via 192.168.114.15, eth4, 00:56:06
     *>                  [20/0] via 192.168.114.14, eth4, 00:56:06
     *>                  [20/0] via 192.168.114.13, eth4, 00:56:06
     *>                  [20/0] via 192.168.114.12, eth4, 00:56:06
     *>                  [20/0] via 192.168.114.11, eth4, 00:56:06
```

&nbsp;



#### ExternalDNS

IPアドレスで接続　やはりドメイン

CoreDNS

etcd

external-dns


```vim
apiVersion: v1
kind: Service
metadata:
  name: nginx-svc-lb
  labels:
    app.kubernetes.io/name: nginx-svc-lb
  annotations:
    external-dns.alpha.kubernetes.io/hostname: nginx.k8s
spec:
  type: LoadBalancer
  ports:
  - protocol: TCP
    port: 80
    targetPort: 80
  selector:
    app.kubernetes.io/name: nginx
```

```bash
$ kubectl get all
NAME                         READY   STATUS    RESTARTS   AGE
pod/nginx-795566ddcd-6wxsd   1/1     Running   0          5h35m

NAME                   TYPE           CLUSTER-IP    EXTERNAL-IP     PORT(S)        AGE
service/kubernetes     ClusterIP      10.96.0.1     <none>          443/TCP        6d13h
service/nginx-svc-lb   LoadBalancer   10.96.26.26   192.168.115.2   80:31333/TCP   5h35m

NAME                    READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/nginx   1/1     1            1           5h35m

NAME                               DESIRED   CURRENT   READY   AGE
replicaset.apps/nginx-795566ddcd   1         1         1       5h35m
```

&nbsp;

キャッシュDNSサーバに対して、ゾーン "*k8s.*" は上述した CoreDNS にフォワードするように設定している。


```bash
$ dig nginx.k8s

; <<>> DiG 9.11.3-1ubuntu1.16-Ubuntu <<>> nginx.k8s
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 996
;; flags: qr rd ra; QUERY: 1, ANSWER: 1, AUTHORITY: 0, ADDITIONAL: 1

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 65494
;; QUESTION SECTION:
;nginx.k8s.			IN	A

;; ANSWER SECTION:
nginx.k8s.		30	IN	A	192.168.115.2

;; Query time: 12 msec
;; SERVER: 127.0.0.53#53(127.0.0.53)
;; WHEN: Tue Feb 08 07:42:20 JST 2022
;; MSG SIZE  rcvd: 54
```

&nbsp;



#### Longhorn

![Longhorn Dashboard](/images/introducing-my-home-lab-in-2022/longhorn_dashboard.png)


&nbsp;



#### InfluxDB

監視サーバ Prometheus の長期記憶ストレージ(LTS)として稼働。

```bash
$ kubectl -n influxdb get all,pvc
NAME                               READY   STATUS    RESTARTS   AGE
pod/influxdb-dp-8447dc855d-hd7tt   2/2     Running   0          3d5h

NAME                               TYPE           CLUSTER-IP       EXTERNAL-IP     PORT(S)          AGE
service/influxdb-exporter-svc-lb   LoadBalancer   10.100.221.224   192.168.115.8   9122:32042/TCP   3d5h
service/influxdb-svc-lb            LoadBalancer   10.105.178.235   192.168.115.7   8086:31480/TCP   3d5h

NAME                          READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/influxdb-dp   1/1     1            1           3d5h

NAME                                     DESIRED   CURRENT   READY   AGE
replicaset.apps/influxdb-dp-8447dc855d   1         1         1       3d5h

NAME                                 STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
persistentvolumeclaim/influxdb-pvc   Bound    pvc-65dfb8bd-40fa-498d-95d5-3e36123be270   128Gi      RWO            longhorn       3d5h
```



&nbsp;



#### sosomasox.com 

本ブログもkubernetes上で



&nbsp;



#### commpost.sosomasox.com

コメント投稿システム。

ブログのフッタに



&nbsp;
