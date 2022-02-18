---
title: "Kubernetes上にGrafanaを導入してInfluxDBに保存されているFitbitデータを可視化してみた"
date: 2020-07-01T22:02:52+09:00
Description: ""
Tags: []
Categories: []
thumbnail: /images/deploying-grafana-on-kubernetes-to-display-fitbit-data-stored-in-influxdb/thumbnail.png
DisableComments: false
---

&nbsp;

先日、[『Kubernetes上にInfluxDBを導入してFitbitデータを保存してみた』](https://techblog.sosomasox.com/posts/store-fitbit-data-to-influxdb-on-kubernetes/)という記事を書きました。

本稿は前回の記事の続きです。

今回、Kubernetes上にGrafanaを導入してInfluxDBに保存されているFitbitデータの心拍数を可視化してみましたのでその紹介をしたいと思います。

&nbsp;



### サービス構成

サービス構成を下図に示します。

ngrokを利用して外部からGrafanaのWebインターフェースにアクセスできるような構成にしました。

Grafanaのダッシュボードをngrokを利用して公開するために ***ngrok-grafana-dp*** というDeploymentリソースを稼働させています。
このリソースはPod内に ***ngrok-http*** コンテナと ***ngrok-proxy*** コンテナの2つのコンテナが稼働しています。

***ngrok-http*** コンテナはngrokサービスに対してHTTPトンネリング接続をおこない、ローカルサービスと結びついた外部公開用URLを取得しています。
もう一方の ***ngrok-proxy*** コンテナは ***ngrok-http*** コンテナが提供しているngrokのWebインターフェースに対するアクセスを可能にするための機能や、ngrokサービスから払い出された外部公開用URLからアクセスされたときにGrafana対してリバースプロキシする機能など **サイドカーコンテナ** としての役割を担っています。

他のサービスとしてgrafana用のDeploymentリソースを稼働させています。
このリソースの内ではダッシュボードを作成するにあたって、[前回の記事](https://techblog.sosomasox.com/posts/store-fitbit-data-to-influxdb-on-kubernetes/)で導入したInfluxDBに対してアクセスをおこなっています。

![Microservices Architecture](/images/deploying-grafana-on-kubernetes-to-display-fitbit-data-stored-in-influxdb/microservices_architecture.jpg)

&nbsp;



### Kubernetesマニフェストファイル

Grafana用のKubernetesマニフェストファイルを下記に示します。

&nbsp;

```:grafana/manifest.yaml
---

apiVersion: apps/v1
kind: Deployment
metadata:
  name: grafana
  namespace: experiment
spec:
  replicas: 1
  selector:
    matchLabels:
      app: grafana
  strategy:
    type: Recreate
  template:
    metadata:
      labels:
        app: grafana
    spec:
      containers:
      - name: grafana
        imagePullPolicy: Always
        image: grafana/grafana:7.0.5
        livenessProbe:
          httpGet:
            path: /api/health
            port: 3000
            scheme: HTTP
          initialDelaySeconds: 180
          periodSeconds: 3
          timeoutSeconds: 3
          successThreshold: 1
          failureThreshold: 3
        ports:
        - containerPort: 3000

---

apiVersion: v1
kind: Service
metadata:
  name: grafana-svc-lb
  namespace: experiment
spec:
  type: LoadBalancer
  ports:
  - protocol: TCP
    port: 3000
    targetPort: 3000
  selector:
    app: grafana

---
```

&nbsp;

ngrok用のKubernetesマニフェストファイルを下記に示します。

&nbsp;

```:ngrok-grafana/manifest.yaml
---

apiVersion: v1
kind: Secret
metadata:
  name: ngrok-grafana-auth-secret
  namespace: experiment
type: Opaque
data:
  ngrok_auth_token: XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX=

---

apiVersion: v1
kind: ConfigMap
metadata:
  name: ngrok-grafana-proxy-cm
  namespace: experiment
data:
  nginx.conf: |
    user  nginx;
    worker_processes  1;

    error_log  /var/log/nginx/error.log warn;
    pid        /var/run/nginx.pid;


    events {
        worker_connections  1024;
    }


    http {
        default_type  application/octet-stream;

        log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
                          '$status $body_bytes_sent "$http_referer" '
                          '"$http_user_agent" "$http_x_forwarded_for"';

        access_log  /var/log/nginx/access.log  main;

        sendfile        on;
        #tcp_nopush     on;

        keepalive_timeout  65;

        #gzip  on;

        include /etc/nginx/conf.d/*.conf;

        server {
            listen 4039;

            location / {
                proxy_pass http://localhost:4040/;
            }

        }

        server {
            listen 8080;

            location / {
                proxy_pass http://grafana-svc-lb:3000/;
            }

        }

    }

---

apiVersion: apps/v1
kind: Deployment
metadata:
  name: ngrok-grafana-dp
  namespace: experiment
  labels:
    app: ngrok-grafana
spec:
  strategy:
    type: Recreate
  replicas: 1
  selector:
    matchLabels:
      app: ngrok-grafana
  template:
    metadata:
      labels:
        app: ngrok-grafana
    spec:
      volumes:
      - name: ngrok-grafana-proxy-nginx-config
        configMap:
          name: ngrok-grafana-proxy-cm
      containers:
      - name: ngrok-proxy
        imagePullPolicy: Always
        image: "izewfktvy533zjmn/ngrok-proxy:latest"
        ports:
        - containerPort: 4039
        volumeMounts:
        - name: ngrok-grafana-proxy-nginx-config
          mountPath: /etc/nginx
          readOnly: true
        livenessProbe:
          httpGet:
            path: /api/tunnels
            port: 4039
            scheme: HTTP
          initialDelaySeconds: 3
          periodSeconds: 1
          timeoutSeconds: 1
          successThreshold: 1
          failureThreshold: 3
      - name: ngrok-http
        imagePullPolicy: Always
        image: "izewfktvy533zjmn/ngrok-http:arm32v7"
        livenessProbe:
          httpGet:
            path: /api/tunnels
            port: 4039
            scheme: HTTP
          initialDelaySeconds: 5
          periodSeconds: 1
          timeoutSeconds: 1
          successThreshold: 1
          failureThreshold: 3
        env:
        - name: NGROK_AUTH_TOKEN
          valueFrom:
            secretKeyRef:
              name: ngrok-grafana-auth-secret
              key: ngrok_auth_token

---

apiVersion: v1
kind: Service
metadata:  
  name: ngrok-grafana-svc-lb
  namespace: experiment
spec:
  type: LoadBalancer
  ports:  
  - protocol: TCP
    port: 4039
    targetPort: 4039
  selector:    
    app: ngrok-grafana

---
```
&nbsp;



### 稼働状況の確認

Grafanaにアクセスするためのエンドポイントとして作成したロードバランサのIPアドレスを取得するために稼働状況を確認します。

&nbsp;

```bash
pi@makina-gateway:~ $ kubectl get all -n experiment
NAME                                         READY   STATUS    RESTARTS   AGE
pod/alexa-skills-dp-669bc988d7-b5slm         1/1     Running   1          4d
pod/fitbit-oauth-dp-5fbdf4d746-7sx8q         1/1     Running   0          4d
pod/grafana-7dcf67c7c5-5xnn9                 1/1     Running   0          7m10s
pod/influxdb-74796cc65-6df8l                 1/1     Running   0          158m
pod/ngrok-alexa-skills-dp-84c84f6d56-jpxzj   2/2     Running   0          4d
pod/ngrok-fitbit-oauth-dp-5b9f8dd9b-nc5nq    2/2     Running   0          3d23h
pod/ngrok-grafana-dp-bf9c97dd6-frp7d         2/2     Running   0          77s

NAME                                TYPE           CLUSTER-IP       EXTERNAL-IP     PORT(S)          AGE
service/alexa-skills-svc-lb         LoadBalancer   10.100.236.34    192.168.3.222   8080:32464/TCP   4d
service/fitbit-oauth-svc-lb         LoadBalancer   10.100.38.241    192.168.3.223   8080:30058/TCP   4d
service/grafana-svc-lb              LoadBalancer   10.110.119.123   192.168.3.227   3000:32520/TCP   7m11s
service/influxdb-svc-lb             LoadBalancer   10.108.30.5      192.168.3.224   8086:31360/TCP   158m
service/ngrok-alexa-skills-svc-lb   LoadBalancer   10.105.34.236    192.168.3.225   4039:31412/TCP   4d
service/ngrok-fitbit-oauth-svc-lb   LoadBalancer   10.98.217.78     192.168.3.226   4039:30616/TCP   3d23h
service/ngrok-grafana-svc-lb        LoadBalancer   10.103.216.118   192.168.3.228   4039:30738/TCP   4m39s

NAME                                    READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/alexa-skills-dp         1/1     1            1           4d
deployment.apps/fitbit-oauth-dp         1/1     1            1           4d
deployment.apps/grafana                 1/1     1            1           7m12s
deployment.apps/influxdb                1/1     1            1           158m
deployment.apps/ngrok-alexa-skills-dp   1/1     1            1           4d
deployment.apps/ngrok-fitbit-oauth-dp   1/1     1            1           3d23h
deployment.apps/ngrok-grafana-dp        1/1     1            1           4m40s

NAME                                               DESIRED   CURRENT   READY   AGE
replicaset.apps/alexa-skills-dp-669bc988d7         1         1         1       4d
replicaset.apps/fitbit-oauth-dp-5fbdf4d746         1         1         1       4d
replicaset.apps/grafana-7dcf67c7c5                 1         1         1       7m12s
replicaset.apps/influxdb-74796cc65                 1         1         1       158m
replicaset.apps/ngrok-alexa-skills-dp-84c84f6d56   1         1         1       4d
replicaset.apps/ngrok-fitbit-oauth-dp-5b9f8dd9b    1         1         1       3d23h
replicaset.apps/ngrok-grafana-dp-bf9c97dd6         1         1         1       4m40s
```

&nbsp;


ロードバランサに対して下記のコマンドを実行することでngrokサービスによって払い出された外部公開用URLを取得します。

```bash
pi@makina-gateway:~ $ curl -s 192.168.3.228:4039/api/tunnels | jq -r .tunnels[].public_url | grep --color=never https://*
https://xxxxxxxxxxxx.ap.ngrok.io
```

&nbsp;



### Grafana ダッシュボードの作成

それではブラウザよりngrokサービスから払い出された外部公開用URLを利用して、GrafanaのWebインターフェースにアクセスしてダッシュボードを作成し、Fitbitのデータを可視化していきます。

まず、アクセスするとログイン画面がでてきます。

&nbsp;

![Login Page on Grafana](/images/deploying-grafana-on-kubernetes-to-display-fitbit-data-stored-in-influxdb/login_page_on_grafana.png)

&nbsp;

ログイン後の画面上で _DATA SOURCES - Add you first data source_ というパネルをクリックしてデータソースを追加していきます。

&nbsp;

![Top Page on Grafana](/images/deploying-grafana-on-kubernetes-to-display-fitbit-data-stored-in-influxdb/top_page_on_grafana.png)

&nbsp;

Time Series databaseとしてInfluxDBを選択します。

&nbsp;

![Add Data Source on Grafana](/images/deploying-grafana-on-kubernetes-to-display-fitbit-data-stored-in-influxdb/add_data_source_on_grafana.png)

&nbsp;

InfluxDBのエンドポイントURLと使用するデータベース名を記入してデータソースを作成します。

&nbsp;

![Data Sources InfluxDB on Grafana](/images/deploying-grafana-on-kubernetes-to-display-fitbit-data-stored-in-influxdb/data_sources_influxdb_on_grafana.png)

&nbsp;

次に、ダッシュボードを作っていきます。
_Add new panel_ をクリックします。

&nbsp;

![Add New Panel on Grafana](/images/deploying-grafana-on-kubernetes-to-display-fitbit-data-stored-in-influxdb/add_new_panel_on_grafana.png)

&nbsp;

あとはてきとうにクエリを発行します。


&nbsp;

![Dashboard on Grafana](/images/deploying-grafana-on-kubernetes-to-display-fitbit-data-stored-in-influxdb/dashboard_on_grafana.png)

&nbsp;

良い感じにFitbitデータの心拍数を可視化できました。

&nbsp;



### まとめ

本稿では、Kubernetes上に導入したGrafanaをngrokサービスによって払い出された外部公開用URLを経由してInfluxDBに保存されているFitbitデータの心拍数を可視化したダッシュボードを作成しました。

今回、実行環境として使用したサービス構成ではGrafanaでダッシュボードを作成しても、そのGrafanaが稼働しているPodが終了した場合にはダッシュボードの情報が失われてしまいます。

運用していくにはGrafanaのデータを管理する構成にしていく必要がありますね。

&nbsp;
