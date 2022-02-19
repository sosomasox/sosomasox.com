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

2022年になって自宅ラボの構成や稼働しているサービスが変わったので紹介。

久々にブログを書いた。


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

LANインタフェースからのDNSクエリは、後述するキャッシュDNSサーバにフォワードするように設定。


#### ポートフォワーディング

WANインターフェスのpppoe0に来たHTTP/HTTPS宛のパケットを後述するラズパイKubernetes上で稼働しているNGINX Ingress Controllerにフォワードするように設定。

本ブログはKubernetes上で運用しているため、この設定で外部公開している。

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



## メトリクス監視基盤

EdgeRouterやDNSサーバ、ラズパイKubernetes用のさーばなど、本環境で稼働しているサーバのメトリクス監視を担う。

Prometheus + 各種Exporter で構築。

![Prometheus Targets](/images/introducing-my-home-lab-in-2022/prometheus_targets.png)

ダッシュボードにはGrafanaを利用。

![Grafana Dashboard](/images/introducing-my-home-lab-in-2022/grafana_dashboard.png)

&nbsp;

AlertmanagerやPushgatewayも構築済み。
(アラートルールをちゃんと整備していきたい)

![Alertmanager Alert](/images/introducing-my-home-lab-in-2022/alertmanager_alert.png)

&nbsp;



## ログ収集基盤

ラズパイKubernetes で稼働しているコンテナのログ収集のための基盤。

現在は主に、本ブログのアクセスログを収集するために使用。


![Kibana Dashborad](/images/introducing-my-home-lab-in-2022/kibana_dashboard.png)

&nbsp;



## Wi-Fiルータ

Wi-Fiルータは [tp-link Wi-Fiルータ Archer AX73](https://www.tp-link.com/jp/home-networking/wifi-router/archer-ax73/) をブリッジモードで動作させ利用している。

スマホやノートPCなど無線機器が接続されてる。


&nbsp;



## ラズパイ Kubernetes


USB接続のSSDを使用したRaspberry Pi 4 Model B 8GB  The Hard Way で構築。

コントロールプレーン3台とノード5台で構成。

Kubernetesのデータストアであるetcd は、コントロールプレーンに併設し、クラスタを構成。

また、kube-apiserver 用のロードバランサは HAPorxy + Keepalived の冗長構成でコントロールプレーンに併設。


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


### [MetalLB](https://github.com/metallb/metallb)

Service リソースで type: LoadBalaner を使用するために導入。

Service リソースにKubernetesクラスタ外の指定したIPアドレスを付与することができるので、Kubernetes クラスタ外からでもKubernetes上で稼働しているサービスにアクセスすることができる。

MetalLBの動作モードはL2モードとBGPモードがあるが、EdgeRouter X がBGPに対応しているのでBGPモードで動作させ、EdgeRouter Xにピアを張っている。

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



### [NGINX Ingreee Controller](https://github.com/kubernetes/ingress-nginx)

Ingressリソースを利用するために導入。

本ブログに対する外部アクセスを制御している。

&nbsp;



### [cert-manger](https://github.com/cert-manager/cert-manager)

kubernetesクラスタ上でSSL/TLS証明書を扱うために導入。

本ブログのドメインに対して、Let's Encryptを利用したSSL証明書の発行を担う。

&nbsp;



### [ExternalDNS](https://github.com/kubernetes-sigs/external-dns)

ServiceリソースやIngressリソースに付与されたKubernetesクラスタ外のIPアドレスにドメインを割り当てるために導入。

CoreDNSを利用してDNSサーバをKubernetes内部に作成し、独自ゾーン"*k8s.*"を管理。

レコードの保存先にはetcdを使用。

キャッシュDNSサーバに対して、ゾーン "*k8s.*" は上述した CoreDNS にフォワードするように設定しているため、自宅ラボ環境にある全ての機器から名前解決が可能。

&nbsp;


#### ちょっとしたデモ

nginxコンテナにServiceリソース。
"test-web.k8s"というドメインを付与。


```vim
apiVersion: v1
kind: Service
metadata:
  name: test-web-svc-lb
  namespace: test-web
  labels:
    app.kubernetes.io/name: test-web-svc-lb
  annotations:
    external-dns.alpha.kubernetes.io/hostname: test-web.k8s
spec:
  type: LoadBalancer
  ports:
  - protocol: TCP
    port: 80
    targetPort: 80
  selector:
    app.kubernetes.io/name: test-web
```

&nbsp;

上記のServiceリソースにはIPアドレス "192.168.115.6"が割り当てられている。

```bash
$ kubectl -n test-web get all
NAME                            READY   STATUS    RESTARTS   AGE
pod/test-web-86cc578bfb-s84gd   1/1     Running   0          14m

NAME                      TYPE           CLUSTER-IP      EXTERNAL-IP     PORT(S)        AGE
service/test-web-svc-lb   LoadBalancer   10.99.241.173   192.168.115.6   80:32736/TCP   14m

NAME                       READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/test-web   1/1     1            1           14m

NAME                                  DESIRED   CURRENT   READY   AGE
replicaset.apps/test-web-86cc578bfb   1         1         1       14m
```

&nbsp;


"test-web.k8s"を名前解決してみると割り当てられていたIPアドレス"192.168.115.6"が返ってきている。

```bash
$ dig test-web.k8s

; <<>> DiG 9.11.3-1ubuntu1.16-Ubuntu <<>> test-web.k8s
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 20763
;; flags: qr rd ra; QUERY: 1, ANSWER: 1, AUTHORITY: 0, ADDITIONAL: 1

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 65494
;; QUESTION SECTION:
;test-web.k8s.			IN	A

;; ANSWER SECTION:
test-web.k8s.		30	IN	A	192.168.115.6

;; Query time: 15 msec
;; SERVER: 127.0.0.53#53(127.0.0.53)
;; WHEN: Sat Feb 19 18:44:10 JST 2022
;; MSG SIZE  rcvd: 57

```

&nbsp;



### [Longhorn](https://github.com/longhorn/longhorn)

ベアメタルKubernetes環境でPersistentVolumeClaimリソースを利用するために導入。

LonghornはKubernetesネイティブの分散型ブロックストレージ。

Webダッシュボードもあって結構使いやすい。

![Longhorn Dashboard](/images/introducing-my-home-lab-in-2022/longhorn_dashboard.png)


&nbsp;



### 自作アプリケーション

ラズパイKubernetes上で稼働している自作アプリケーション。


#### [sosomasox.com](https://sosomasox.com)

本ブログ。


#### [commpost.sosomasox.com](https://commpost.sosomasox.com)

本ブログのコメント投稿システム。

ブログにコメント投稿機能を追加した。

↓ COMMPOSTについて昔書いた記事

- [コメント投稿システム『COMMPOST』の紹介](https://sosomasox.com/posts/commpost-a-comment-posting-system/)
- [TECHブログのHUGOテーマを拡張してコメント投稿システムである『COMMPOST』を利用できるようにした話](https://sosomasox.com/posts/expanded-the-hugo-theme-on-the-tech-blog-to-enable-the-commpost-which-is-comment-posting-system-to-be-used/)

&nbsp;



# その他

本ブログで使用しているドメイン "sosomasox.com" はさくらのクラウド DNSアプライアンスを利用して管理している。

ドメイン "sosomasox.com" のメールサーバはさくらのレンタルサーバを利用しており、さくらのクラウド DNSアプライアンスで管理されているMXレコードをレンタルサーバに向けている。

&nbsp;



# 今後取り組みたいこと

- メトリクス監視基盤のアラート整備
- ログ収集基盤の充実化
- (K8sで稼働するなんかしらの)Webアプリの制作 & CI/CD環境の構築

&nbsp;



# 最後に

この辺もう少し詳しく聞きたいよってのがあったら、コメント下さい。

&nbsp;
