---
title: "コメント投稿システム『COMMPOST』の紹介"
date: 2020-07-31T20:12:49+09:00
Description: ""
thumbnail: /images/commpost-a-comment-posting-system/thumbnail.png
Tags: []
Categories: []
DisableComments: false
---

&nbsp;

このブログの運営を始めてから2ヶ月弱経過しました。

本ブログに何か新しい機能を追加したいと考え、すぐに思いついたのがコメント投稿機能でした。

本ブログは[HUGO](https://gohugo.io/)という静的サイトジェネレータを用いて作成を行っているのですが、HUGOによって作成されたブログでは[DISQUS](https://disqus.com/)というコメント投稿サービスを用いることで容易にコメント機能を追加することができます。

しかし、今回、コメント投稿機能を追加するにあたってDISQUSは使用せずに独自のコメント投稿システムを開発し、運用していこうと考えました。

本稿では、本ブログにおいてコメント投稿機能を追加するために開発したコメント投稿システム[『COMMPOST』](https://commpost.sosomasox.com)を紹介していきたいと思います。

![COMMPOST LOGO](/images/commpost-a-comment-posting-system/logo_icon.png)

&nbsp;





## システム要件と構成

### 要件定義
このコメント投稿システムを開発するにあたって、考慮した点を下記にまとめました。


#### 機能要件
- コメントを投稿することができる
- コメントを取得することができる
- コメントを削除することができる
- 特定の記事に対するコメントを取得することができる

コメント投稿システムに最低限必要な機能は有しているかと思われます。


#### 非機能要件
- Kubernetes上で稼働させること
- ヘルスチェック機能を有すること
- XSSへの対策をすること

本システムはこのブログと同様、Kubernetesクラスター上で運用することを想定しています。
また、基本的にこのコメント投稿システムを利用する本ブログは誰でもアクセスするとこができるため、XSS攻撃への対策は必須かと思われます。


#### 実装しないこと
- ユーザ登録やユーザ認証
- コメントに対する返信コメント機能
- Webサービスのような機能提供

DISQUSのようなWebサービスの開発・運用は行いません(行なえませんでした)。

&nbsp;



### システム構成
本システムの構成を下図に示します。

バックエンドにはコメント投稿機能を提供するAPIサーバを設置し、データベースにはMongoDBを利用しました。

フロントエンドとしてブログ記事にスクリプトを組み込んでおき、ブログ記事が読み込まれた際にはコメント取得のAPIリクエストを送るスクリプトを実行することでコメントを取得し、コメントを表示することができるような仕組みです。また、コメントを投稿する際にはコメント投稿のAPIリクエストを送るスクリプトを実行することでコメントの投稿を行うことができます。

&nbsp;

![Constitution](/images/commpost-a-comment-posting-system/constitution.jpg)

&nbsp;





## データベース

### MongoDBのスキーマデザイン
ブログ記事にコメント投稿するために最低限必要となるデータをモデル化しました。

*__"article_id"__* はあるブログ記事に対応する固有のIDであり、投稿されたコメントがどの記事のものなのか識別するために使用されます。
*__"poster_name"__* はコメントの投稿者の名前、*__"text"__* はコメントそのものです。
*__"date"__* はコメントを投稿した時刻を表します。

![Schema Design](/images/commpost-a-comment-posting-system/schema_design.jpg)

&nbsp;





## バックエンド
バックエンドのAPIサーバは[こちらのGitHubリポジトリ](https://github.com/izewfktvy533zjmn/commpost)上で開発を行っています。



### APIリファレンス
COMMPOSTのAPIリファレンスは[こちらのGitHub Wiki](https://github.com/izewfktvy533zjmn/commpost/wiki/Web-API-v1-:-Comments)にまとめました。



### Kubernetes上でのサービス構成
Kubernetes上でのサービス構成を下図に示します。

本システムはMongoDBとAPIサーバををKuberentes上で動作させています。
ngrokサービスを利用することで、外部からCOMMPOSTのAPIサーバにアクセスできるような構成となっています。

&nbsp;

![Microservices Architecture](/images/commpost-a-comment-posting-system/microservices_architecture.jpg)

&nbsp;



### Kubernetesマニフェストファイル
上記の構成図の中からMongoDBとAPIサーバを稼働させているKubernetesマニフェストを紹介します。

MongoDBを稼働させているKubernetesマニフェストファイルは下記の通りです。

&nbsp;

```:mongodb/manifest.yaml
---

apiVersion: v1
kind: PersistentVolume
metadata:
  name: mongodb-pv
  labels:
    volume: mongodb-pv
spec:
  capacity:
    storage: 16Gi
  accessModes:
  - ReadWriteMany
  nfs:
    server: 192.168.11.100
    path: /srv/nfs/on-going/mongodb

---

apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mongodb-pvc
  namespace: on-going
spec:
  accessModes:
  - ReadWriteMany
  resources:
    requests:
      storage: 16Gi
  selector:
    matchLabels:
      volume: mongodb-pv

---

apiVersion: apps/v1
kind: Deployment
metadata:
  name: mongodb-dp
  namespace: on-going
spec:
  replicas: 1
  selector:
    matchLabels:
      app: mongodb
  strategy:
    type: Recreate
  template:
    metadata:
      labels:
        app: mongodb
    spec:
      containers:
      - name: mongodb
        imagePullPolicy: Always
        image: mongo:4.2.8-bionic
        ports:
        - containerPort: 27017
        volumeMounts:
        - name: mongodb-volume
          mountPath: /data/db
      volumes:
      - name: mongodb-volume
        persistentVolumeClaim:
          claimName: mongodb-pvc

---

apiVersion: v1
kind: Service
metadata:
  name: mongodb-svc-lb
  namespace: on-going
spec:
  type: LoadBalancer
  ports:
  - protocol: TCP
    port: 27017
    targetPort: 27017
  selector:
    app: mongodb

---
```

&nbsp;

APIサーバをMongoDBを稼働させているKubernetesマニフェストファイルは下記の通りです。

&nbsp;

```:commpost-api/manifest.yaml
---

apiVersion: v1
kind: ConfigMap
metadata:
  name: commpost-api-cm
  namespace: on-going
data:
  endpoint: mongodb://mongodb-svc-lb:27017/commpost

---

apiVersion: apps/v1
kind: Deployment
metadata:
  name: commpost-api-dp
  namespace: on-going
  labels:
    app: commpost-api
spec:
  replicas: 1
  selector:
    matchLabels:
      app: commpost-api
  template:
    metadata:
      labels:
        app: commpost-api
    spec:
      containers:
      - name: commpost-api
        imagePullPolicy: Always
        image: "izewfktvy533zjmn/commpost:arm64"
        livenessProbe:
          httpGet:
            path: /api/v1/healthz
            port: 3000
            scheme: HTTP
          initialDelaySeconds: 30
          periodSeconds: 3
          timeoutSeconds: 3
          successThreshold: 1
          failureThreshold: 3
        env:
        - name: ENDPOINT
          valueFrom:
            configMapKeyRef:
              name: commpost-api-cm
              key: endpoint

---

apiVersion: v1
kind: Service
metadata:
  name: commpost-api-svc-lb
  namespace: on-going
spec:
  type: LoadBalancer
  ports:
  - protocol: TCP
    port: 3000
    targetPort: 3000
  selector:
    app: commpost-api

---
```

&nbsp;





## フロントエンド
フロントエンドは投稿されたコメントの表示とコメントを投稿するフォームを提供します。

下記のHTMLをブログ記事に組み込むことでコメント投稿システム『COMMPOST』を利用することができます。

&nbsp;

```:commpost.html
<div id="commpost"></div>
<script type="text/javascript">
    (function() {
        var dsq = document.createElement('script');
        dsq.type = 'text/javascript'; dsq.async = true; dsq.src = 'https://commpost.sosomasox.com/js/embed.js';
        (document.getElementsByTagName('head')[0] || document.getElementsByTagName('body')[0]).appendChild(dsq);

         var dlq = document.createElement('link');
         dlq.rel = 'stylesheet'; dlq.type = 'text/css'; dlq.href = 'https://commpost.sosomasox.com/assets/css/embed.css';
         (document.getElementsByTagName('head')[0] || document.getElementsByTagName('body')[0]).appendChild(dlq);
    })();
</script>
<link href="https://fonts.googleapis.com/css2?family=Fredoka+One&display=swap" rel="stylesheet">
```

&nbsp;

実はまだ、フロントエンド側は開発途中です。

というのも筆者はフロントエンド側の開発の経験がないので、試行錯誤しながら開発を進めています。

HTML + CSS + JavaScriptの勉強を兼ねて取り組んでいる状況です。


&nbsp;





## 最後に
現状で稼働しているコメント投稿システム『COMMPOST』の機能を本ブログ記事の最後に組み込みました。

今後はフロントエンド側のデザインを整えていきます。

また、今回開発したコメント投稿システム『COMMPOST』を本ブログで安易に利用できるように、本ブログで使用しているHUGOのテンプレートを拡張したいと思っています。

&nbsp;
