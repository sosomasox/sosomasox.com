---
title: "Techブログはじめます！"
date: 2020-05-12T21:54:40+09:00
Description: ""
thumbnail: /images/starting-a-tech-blog/physical_raspi_kubernetes_cluster.jpg
Tags: []
Categories: []
DisableComments: false
---

&nbsp;

この度、自宅プライベートクラウド上でTechブログの運用を行うことにしました。

しばらくの間、Techブログとして[Qiita](https://qiita.com/sosomasox)を利用しておりましたが、先月構築した[高可用性ラズパイKubernetesクラスター](https://qiita.com/izewfktvy533/items/efaa0c5fd6b6ea2c691b)を自宅プライベートクラウドとして運用していく上で、何らかのサービスがあるとモチベーションの維持に繋がると考え、このブログを一種のサービスとて運用していきたいと思います。

&nbsp;

![Physical Raspberry Pi Kubernetes Cluster](/images/starting-a-tech-blog/physical_raspi_kubernetes_cluster.jpg)

&nbsp;



### このブログで取り上げる内容
本プログでは、下記の内容に関して取り上げていきたいと考えています。

- ラズパイKubernetesクラスタを基盤とした自宅プライベートクラウド構築プロジェクトの紹介
- 自宅プライベートクラウド上で稼働しているサービスの運用方法やシステムアーキテクチャーの紹介
- コンピュータ・サイエンスやソフトウェアエンジニアリングに関すること

&nbsp;


### このブログの継続的デリバリー
本ブログで継続的デリバリーを行うために、下図のような構成を採っています。

&nbsp;

![Tech Blog CD Overvies](/images/starting-a-tech-blog/on-going_cd_overview.png)

&nbsp;

HUGOとGitを利用してブログ記事の作成と管理をしており、GitHubリポジトリ上のmasterブランチへのpushをトリガーとして継続的デリバリーが行われる仕組みになっています。

もちろんサービスはコンテナとして運用しています。

&nbsp;

![Kubernetes Pods in ongoing namespace](/images/starting-a-tech-blog/on-going.png)

&nbsp;



### 最後に
記事に対するコメントや感想はTwitterでお待ちしています。

読者になっていただけると幸いです。

&nbsp;
