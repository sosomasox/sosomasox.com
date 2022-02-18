---
title: "Kubernetes上にInfluxDBを導入してFitbitデータを保存してみた"
date: 2020-06-26T18:29:15+09:00
Description: ""
Tags: []
Categories: []
thumbnail: images/store-fitbit-data-to-influxdb-on-kubernetes/thumbnail.jpg
DisableComments: false
---

&nbsp;

今回、Kubernetes上に導入したInfluxDBに対してFitbitデータの保存を試みましたので、本稿でその紹介をしたいと思います。

余談ですが筆者はFitbitユーザであり、使い始めて今年で3年目になります。

今後、これまでにFitbitデバイスで蓄積してきたデータを今回構築した環境上で分析できたらと思います。

&nbsp;



### Kubernetes上にInfluxDBを導入する

[InfluxDB](https://www.influxdata.com/)とはInfluxData社が開発・提供している[オープンソース](https://github.com/influxdata/influxdb)の時系列データベースで、サーバのメトリクスやアプリケーションログ、IoTシステムなどから得られたセンシングデータなどの時系列データの取扱に特化したデータベースです。

[![influxdb logo](images/store-fitbit-data-to-influxdb-on-kubernetes/Influxdb_logo-960x504.png)](https://www.influxdata.com/)

この時系列データベースであるInfluxDBをRaspberry Piで構築したKubernetes上にデプロイします。
InfluxDBのDockerイメージにはRaspberry Piで動作可能な[arm32v7/influxdb](https://hub.docker.com/r/arm32v7/influxdb)を使用しました。

WebAPIエンドポイントに対する認証設定やHTTPS化などは行っておりません。

早速ですが、Kubernetesマニフェストファイルを下記に示します。

&nbsp;

```yaml
---

apiVersion: v1
kind: PersistentVolume
metadata:
  name: influxdb-nfs-pv
  labels:
    volume: influxdb-nfs-pv
spec:
  capacity:
    storage: 16Gi
  accessModes:
  - ReadWriteMany
  nfs:
    server: 192.168.3.241
    path: /srv/nfs/experiment/influxdb

---

apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: influxdb-nfs-pvc
  namespace: experiment
spec:
  accessModes:
  - ReadWriteMany
  resources:
    requests:
      storage: 16Gi
  selector:
    matchLabels:
      volume: influxdb-nfs-pv

---

apiVersion: apps/v1
kind: Deployment
metadata:
  name: influxdb
  namespace: experiment
spec:
  replicas: 1
  selector:
    matchLabels:
      app: influxdb
  strategy:
    type: Recreate
  template:
    metadata:
      labels:
        app: influxdb
    spec:
      containers:
      - name: influxdb
        image: arm32v7/influxdb:1.8.0
        livenessProbe:
          httpGet:
            path: /health
            port: 8086
            scheme: HTTP
          initialDelaySeconds: 15
          periodSeconds: 3
          timeoutSeconds: 3
          successThreshold: 1
          failureThreshold: 3
        ports:
        - containerPort: 8086
        volumeMounts:
        - name: influxdb-volume
          mountPath: /var/lib/influxdb
      volumes:
      - name: influxdb-volume
        persistentVolumeClaim:
          claimName: influxdb-nfs-pvc

---

apiVersion: v1
kind: Service
metadata:
  name: influxdb-svc-lb
  namespace: experiment
spec:
  type: LoadBalancer
  ports:
  - protocol: TCP
    port: 8086
    targetPort: 8086
  selector:
    app: influxdb

---
```

&nbsp;

InfluxDB用のボリュームポイントとしてPersistentVolumeリソースとPersistentVolumeClaimリソースを作成し、提供しています。

InfluxDBには[ヘルスチェックのためのエンドポイント(/health)](https://docs.influxdata.com/influxdb/v1.8/tools/api/#health-http-endpoint)が用意されています。本環境ではKubernetesのヘルスチェック機能の一つであるLivenessProbeを用いてInfluxDBに対してヘルスチェックを行うようにしました。

また、InfluxDBにアクセスするためのエンドポイントを提供するために、ロードバランサの払い出しを行っています。

このマニフェストファイルをKubernetesに適用した結果を下記に示します。

&nbsp;

```bash
pi@makina-gateway:~/workspace/experiment/manifests/influxdb $ kubectl apply -f .
persistentvolume/influxdb-nfs-pv created
persistentvolumeclaim/influxdb-nfs-pvc created
deployment.apps/influxdb created
service/influxdb-svc-lb created
pi@makina-gateway:~/workspace/experiment/manifests/influxdb $ kubectl -n experiment get all
NAME                                         READY   STATUS    RESTARTS   AGE
pod/alexa-skills-dp-669bc988d7-gjn2b         1/1     Running   0          6d17h
pod/fitbit-oauth-dp-6dd8b4b9d-z4xz5          1/1     Running   0          6d18h
pod/influxdb-bf655dc8b-c7wl7                 1/1     Running   0          3m2s
pod/ngrok-alexa-skills-dp-84c84f6d56-pm6cr   2/2     Running   0          6d17h
pod/ngrok-fitbit-oauth-dp-5b9f8dd9b-q6j7z    2/2     Running   2          6d18h

NAME                                TYPE           CLUSTER-IP       EXTERNAL-IP     PORT(S)          AGE
service/alexa-skills-svc-lb         LoadBalancer   10.105.183.248   192.168.3.223   8080:30475/TCP   6d17h
service/fitbit-oauth-svc-lb         LoadBalancer   10.98.184.226    192.168.3.222   8080:31697/TCP   6d18h
service/influxdb-svc-lb             LoadBalancer   10.98.198.103    192.168.3.226   8086:32192/TCP   3m2s
service/ngrok-alexa-skills-svc-lb   LoadBalancer   10.103.140.123   192.168.3.225   4039:30877/TCP   6d17h
service/ngrok-fitbit-oauth-svc-lb   LoadBalancer   10.105.225.110   192.168.3.224   4039:31057/TCP   6d18h

NAME                                    READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/alexa-skills-dp         1/1     1            1           6d17h
deployment.apps/fitbit-oauth-dp         1/1     1            1           6d18h
deployment.apps/influxdb                1/1     1            1           3m3s
deployment.apps/ngrok-alexa-skills-dp   1/1     1            1           6d17h
deployment.apps/ngrok-fitbit-oauth-dp   1/1     1            1           6d18h

NAME                                               DESIRED   CURRENT   READY   AGE
replicaset.apps/alexa-skills-dp-669bc988d7         1         1         1       6d17h
replicaset.apps/fitbit-oauth-dp-6dd8b4b9d          1         1         1       6d18h
replicaset.apps/influxdb-bf655dc8b                 1         1         1       3m3s
replicaset.apps/ngrok-alexa-skills-dp-84c84f6d56   1         1         1       6d17h
replicaset.apps/ngrok-fitbit-oauth-dp-5b9f8dd9b    1         1         1       6d18h
```

&nbsp;

Kubernetesのヘルスチェック機能であるLivenessProbeが動作していることを確認してみます。

&nbsp;

```bash
pi@makina-gateway:~ $ kubectl -n experiment logs --tail=3 pod/influxdb-bf655dc8b-c7wl7 
[httpd] 10.244.7.1 - - [26/Jun/2020:12:11:21 +0000] "GET /health HTTP/1.1" 200 106 "-" "kube-probe/1.13" 2615250d-b7a6-11ea-82c0-8a2dcd57b427 166
[httpd] 10.244.7.1 - - [26/Jun/2020:12:11:24 +0000] "GET /health HTTP/1.1" 200 106 "-" "kube-probe/1.13" 27deeb19-b7a6-11ea-82c1-8a2dcd57b427 214
[httpd] 10.244.7.1 - - [26/Jun/2020:12:11:27 +0000] "GET /health HTTP/1.1" 200 106 "-" "kube-probe/1.13" 29a8a504-b7a6-11ea-82c2-8a2dcd57b427 165
```
&nbsp;

InfluxDBにはインスタンスの状態ととバーションを確認するためのエンドポイントとして[/ping](https://docs.influxdata.com/influxdb/v1.8/tools/api/#ping-http-endpoint)エンドポイントが用意されています。
Kubernetes上にデプロイしたInfluxDBに対して稼働しているかこのエンドポイントを用いて確認してみます。

&nbsp;

```bash
pi@makina-gateway:~ $ curl -sl -I http://192.168.3.226:8086/ping?verbose=true
HTTP/1.1 200 OK
Content-Type: application/json
Request-Id: ac114c50-b7a2-11ea-80c0-8a2dcd57b427
X-Influxdb-Build: OSS
X-Influxdb-Version: 1.8.0
X-Request-Id: ac114c50-b7a2-11ea-80c0-8a2dcd57b427
Date: Fri, 26 Jun 2020 11:46:28 GMT
Content-Length: 19

```

&nbsp;

レスポンスが返ってきており、ちゃんと稼働していることが確認できました。

次に、データベースを作成して、データを保存してみます。

&nbsp;

```bash
pi@makina-gateway:~ $ curl -XPOST 'http://192.168.3.226:8086/query' --data-urlencode 'q=CREATE DATABASE "mydb"'
{"results":[{"statement_id":0}]}
pi@makina-gateway:~ $ curl -i -XPOST 'http://192.168.3.226:8086/write?db=mydb' --data-binary 'cpu_load_short,host=server01,region=us-west value=0.64 1434055562000000000'
HTTP/1.1 204 No Content
Content-Type: application/json
Request-Id: 4e2d1992-b7a4-11ea-81b4-8a2dcd57b427
X-Influxdb-Build: OSS
X-Influxdb-Version: 1.8.0
X-Request-Id: 4e2d1992-b7a4-11ea-81b4-8a2dcd57b427
Date: Fri, 26 Jun 2020 11:58:10 GMT

pi@makina-gateway:~ $ curl -i -XPOST 'http://192.168.3.226:8086/write?db=mydb' --data-binary 'cpu_load_short,host=server01,region=us-west value=0.74 1434055622000000000'
HTTP/1.1 204 No Content
Content-Type: application/json
Request-Id: afe181f0-b7a4-11ea-81eb-8a2dcd57b427
X-Influxdb-Build: OSS
X-Influxdb-Version: 1.8.0
X-Request-Id: afe181f0-b7a4-11ea-81eb-8a2dcd57b427
Date: Fri, 26 Jun 2020 12:00:54 GMT

pi@makina-gateway:~ $ curl -i -XPOST 'http://192.168.3.226:8086/write?db=mydb' --data-binary 'cpu_load_short,host=server01,region=us-west value=0.84 1434055682000000000'
HTTP/1.1 204 No Content
Content-Type: application/json
Request-Id: bdbdc90b-b7a4-11ea-81f4-8a2dcd57b427
X-Influxdb-Build: OSS
X-Influxdb-Version: 1.8.0
X-Request-Id: bdbdc90b-b7a4-11ea-81f4-8a2dcd57b427
Date: Fri, 26 Jun 2020 12:01:17 GMT

```

&nbsp;

データが保存されていることを確認してみます。

&nbsp;

```bash
pi@makina-gateway:~ $ curl -i -XPOST 'http://192.168.3.226:8086/query?db=mydb&pretty=true' --data-urlencode 'q=SELECT * FROM "cpu_load_short"'
HTTP/1.1 200 OK
Content-Type: application/json
Request-Id: fe7996fd-b7a4-11ea-821a-8a2dcd57b427
X-Influxdb-Build: OSS
X-Influxdb-Version: 1.8.0
X-Request-Id: fe7996fd-b7a4-11ea-821a-8a2dcd57b427
Date: Fri, 26 Jun 2020 12:03:06 GMT
Transfer-Encoding: chunked

{
    "results": [
        {
            "statement_id": 0,
            "series": [
                {
                    "name": "cpu_load_short",
                    "columns": [
                        "time",
                        "host",
                        "region",
                        "value"
                    ],
                    "values": [
                        [
                            "2015-06-11T20:46:02Z",
                            "server01",
                            "us-west",
                            0.64
                        ],
                        [
                            "2015-06-11T20:47:02Z",
                            "server01",
                            "us-west",
                            0.74
                        ],
                        [
                            "2015-06-11T20:48:02Z",
                            "server01",
                            "us-west",
                            0.84
                        ]
                    ]
                }
            ]
        }
    ]
}
```

&nbsp;

Kubernetes上でInfluxDBが正常に稼働していることが確認できました。

&nbsp;



### FitbitのデータをInfluxDBに保存する

それではFitbitのデータをInfluxDBに保存していきます。

まず、あらかじめ下記のコマンドを実行してInfluxDB上にデータベースを作成します。

&nbsp;

```bash
pi@makina-gateway:~ $ curl -XPOST 'http://192.168.3.226:8086/query' --data-urlencode 'q=CREATE DATABASE "fitbit"'
{"results":[{"statement_id":0}]}
```

&nbsp;

Fitbit APIを利用してFitbitデータの取得行い、取得したデータをInfluxDBに保存するためのプログラムを下記に示します。

プログラム上でFitbitのAPIを利用するため、認証に必要なアクセストークンなどは事前に用意ました。
また、Pythonパッケージに[*fitbit*](https://pypi.org/project/fitbit/)と[*influxdb*](https://pypi.org/project/influxdb/)を利用しています。

&nbsp;

```:store_fitbit_data_to_influxdb.py
#!/usr/env/bin python3

import ast
import datetime
import fitbit
from influxdb import InfluxDBClient

def update_token(token):
    f = open(TOKEN_FILE, 'w')
    f.write(str(token))
    f.close()

CLIENT_ID     = "XXXXXX"
CLIENT_SECRET = "YYYYYYYYYYYYYYYYYYYYYYYYYYYYYYYY"
TOKEN_FILE    = "token.txt"

token = open(TOKEN_FILE).read()
token_dict = ast.literal_eval(token)
ACCESS_TOKEN = token_dict['access_token']
REFRESH_TOKEN = token_dict['refresh_token']

client_fitbit = fitbit.Fitbit(client_id=CLIENT_ID, 
                              client_secret=CLIENT_SECRET,
                              access_token=ACCESS_TOKEN, 
                              refresh_token=REFRESH_TOKEN, 
                              refresh_cb=update_token)

today_datetime  = datetime.date.today() - datetime.timedelta(days=0)

_data  = client_fitbit.intraday_time_series('activities/heart', 
                                            today_datetime, 
                                            detail_level='1sec')

heartrate_lst = _data["activities-heart-intraday"]["dataset"]

client_influxdb = InfluxDBClient(host='192.168.3.226', port=8086, database='fitbit')

for heartrate in heartrate_lst:
    date_str = today_datetime.strftime('%Y-%m-%d') + " " + str(heartrate['time'])
    date_datetime = datetime.datetime.strptime(date_str, '%Y-%m-%d %H:%M:%S')

    _data = [
            {
                "measurement": "test",
                "time": date_datetime,
                "fields": {
                    "heart_rate": heartrate['value']
                    }
                }
            ]

    try:
        result = client_influxdb.write_points(_data)
        print(_data, result)

        if not result:
            exit(1)

    except Exception:
            exit(1)

exit(0)
```

&nbsp;

上記のプログラム実行後、FitbitデータがInfluxDBに保存されているか確認します。

&nbsp;

```bash
pi@makina-gateway:~ $ curl -i -XPOST 'http://192.168.3.224:8086/query?db=fitbit&pretty=true' --data-urlencode 'q=SELECT * FROM "test"'
HTTP/1.1 200 OK
Content-Type: application/json
Request-Id: cb767efd-b850-11ea-9885-aa7a00487a86
X-Influxdb-Build: OSS
X-Influxdb-Version: 1.8.0
X-Request-Id: cb767efd-b850-11ea-9885-aa7a00487a86
Date: Sat, 27 Jun 2020 08:32:53 GMT
Transfer-Encoding: chunked

{
    "results": [
        {
            "statement_id": 0,
            "series": [
                {
                    "name": "test",
                    "columns": [
                        "time",
                        "heart_rate"
                    ],
                    "values": [
                        [
                            "2020-06-27T00:00:01Z",
                            79
                        ],
                        [
                            "2020-06-27T00:00:11Z",
                            77
                        ],
                        [
                            "2020-06-27T00:00:21Z",
                            76
                        ],

```

&nbsp;

上記に示した結果より、InfluxDBにFitbitデータが保存されていることが確認できました。

&nbsp;



### まとめ

本稿では、Kubernetes上に導入したInfluxDBに対して、Fitbit APIを利用して取得したFitbitデータを保存してみた取り組みを紹介しました。

今後は、Grafanaを利用したFitbitデータの可視化やKubernetesのCronJobリソースを利用したFitbitデータ収集基盤の構築、データ分析基盤の構築などを行っていきたいと考えています。
