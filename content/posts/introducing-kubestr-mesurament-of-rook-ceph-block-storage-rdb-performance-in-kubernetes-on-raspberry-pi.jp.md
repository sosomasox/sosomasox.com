---
title: "kubestrでラズパイKubernetesクラスター上のRook/Cephで作成したブロックストレージ(RBD)に対してパフォーマンス測定してみる"
date: 2021-04-04T22:05:38+09:00
Description: ""
Tags: []
Categories: []
thumbnail: /images/introducing-kubestr-mesurament-of-rook-ceph-block-storage-rdb-performance-in-kubernetes-on-raspberry-pi/thumbnail.png
DisableComments: false
---

&nbsp;

[kubenews #14](https://www.youtube.com/watch?v=VxRDMBmaDgU&t=2135s)で紹介されていた[kubestr](https://kubestr.io/)というツールを紹介していたのでラズパイKubernetesクラスター上で使用してみました。

&nbsp;

![Thumbnail](images/introducing-kubestr-mesurament-of-rook-ceph-block-storage-rdb-performance-in-kubernetes-on-raspberry-pi/thumbnail.png)

&nbsp;

[kubestr](https://kubestr.io/)はKubernetesでストレージを使用するためにストレージの発見、検証、評価をおこなうことが可能なツールです。

kubestrを使用することで以下のようなことができます。

- Kubernetesクラスター上に存在する様々なストレージを特定することができる
- ストレージが正しく設定されているかどうか検証することができる
- [FIO](https://github.com/axboe/fio)といったベンチマークツールを使用し、ストレージ性能を評価することができる

&nbsp;

kubestrは現状(2021年4月4日現在)、arm64アーキテクチャーに対応したリリースがないので自身でarm64用のビルドを行わないとRaspberry Piなどでは使用できません。

本稿では、kubestrをRaspberry Piで使用するためにarm64用にビルドし、ラズパイKubernetesクラスター上のRook/Cephで作成したブロックストレージ(RBD)に対してkubestrを使用してパフォーマンス測定をしてみます。

&nbsp;



### kubestrをarm64用にビルドする
まずはkubestrがfioを実行するために使用するPodのイメージをビルドしていきます。

下記のコマンドを実行し、kubestrをクローンします。

```bash
git clone https://github.com/kastenhq/kubestr.git
cd kubestr
```

&nbsp;

Dockerfileを編集します。

下記のように **GOARCH=amd64** を **GOARCH=arm64** に変更します。

```bash
$ git diff
diff --git a/Dockerfile b/Dockerfile
index 34bb921..cac1ae1 100644
--- a/Dockerfile
+++ b/Dockerfile
@@ -3,7 +3,7 @@ FROM golang:alpine3.12 AS builder
 ENV GO111MODULE=on \
     CGO_ENABLED=0 \
     GOOS=linux \
-    GOARCH=amd64 \
+    GOARCH=arm64 \
     GOBIN=/dist

 WORKDIR /app
```

&nbsp;

Dockerイメージをビルドし、自身のDockerHubリポジトリにプッシュします。

**{your_dockerhub_id}** には自身のDockerHubのIDを入れて下さい。

```bash
docker build -t kubestr:arm64 .
docker tag kubestr:arm64 {your_dockerhub_id}/kubestr:arm64
docker push {your_dockerhub_id}/kubestr:arm64
```

&nbsp;

次はkubestrコマンドをビルドしていきます。

pkg/common/common.go を下記のように編集します。

**DefaultPodImage** を先程作成したPodのイメージをプッシュした自身のDockerHubリポジトリ先に変更します。
　
```bash
$ git diff
diff --git a/pkg/common/common.go b/pkg/common/common.go
index cbd966c..ac5263f 100644
--- a/pkg/common/common.go
+++ b/pkg/common/common.go
@@ -8,7 +8,7 @@ const (
        // VolSnapClassStableDriverKey describes the stable driver key
        VolSnapClassStableDriverKey = "driver"
        // DefaultPodImage the default pod image
-       DefaultPodImage = "ghcr.io/kastenhq/kubestr:latest"
+       DefaultPodImage = "{your_dockerhub_id}/kubestr:arm64"
        // SnapGroupName describes the snapshot group name
        SnapGroupName = "snapshot.storage.k8s.io"
        // VolumeSnapshotClassResourcePlural  describes volume snapshot classses
```

&nbsp;

最後にkubestrコマンをビルドします。

```bash
go build -o kubestr main.go
```

&nbsp;

kubestrコマンドを実行できるか確認します。

下図は筆者の環境上での実行結果です。

![kubestr](images/introducing-kubestr-mesurament-of-rook-ceph-block-storage-rdb-performance-in-kubernetes-on-raspberry-pi/kubestr.png)

&nbsp;



### kubestrを使用したブロックストレージ(RBD)に対してパフォーマンス測定

kubestrを使用したストレージのパフォーマンス測定は下記のコマンドで実行できます。

```bash
./kubestr fio -s <storage class>
```

下図は筆者の環境上での実行結果です。

![kubestr_fio_rbd](images/introducing-kubestr-mesurament-of-rook-ceph-block-storage-rdb-performance-in-kubernetes-on-raspberry-pi/kubestr_fio_rbd.png)

&nbsp;



### 最後に
本稿では、kubestrをarm64アーキテクチャー用にビルドし、kubestrを使用したブロックストレージ(RBD)に対してパフォーマンス測定を行いました。



&nbsp;
