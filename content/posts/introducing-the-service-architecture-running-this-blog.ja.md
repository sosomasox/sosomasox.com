---
title: "本ブログを運営しているサービスアーキテクチャーの紹介"
date: 2020-05-30T15:06:15+09:00
Description: ""
thumbnail: /images/introducing-the-service-architecture-running-this-blog/service_architecture.jpg
Tags: []
Categories: []
DisableComments: false
---

&nbsp;

先日、[本ブログを運用している自宅インフラの紹介](http://techblog.sosomasox.com/posts/introducing-the-architecture-of-raspberry-pi-kubernetes-cluster-named-makina-that-operates-this-blog/)を行いました。
本稿は、このブログを運営するために自宅インフラ上で稼働しているサービスとそのアーキテクチャーを紹介したいと思います。

&nbsp;



### サービスアーキテクチャー
すでにいくつかのブログ記事で言及しておりますが、本ブログはRaspberry Piで構築した高可用性Kuberntesクラスターである自宅インフラ上で運用しています。

本ブログを運営するにあたって自宅インフラ上で稼働しているKubernetesリソース群とそれらの構成をまとめたサービスアーキテクチャーを下図に示します。



&nbsp;

![Service Architecture](/images/introducing-the-service-architecture-running-this-blog/service_architecture.jpg)

&nbsp;

本サービスアーキテクチャーは大きく4つのコンポーネントで構成されており、これらの連携によって本ブログの運営が可能となっています。

以降では、それぞれのコンポーネントの役割や構成を説明していきたいと思います。

&nbsp;

![Deformation Service Architecture](/images/introducing-the-service-architecture-running-this-blog/deformation_service_architecture.jpg)

&nbsp;



### ongoing
ますはじめに、このコンポーネントは本ブログを公開するwebサーバとしての役割を担っています。
nginxのPodが3つ稼働しており、LoadBalancer Serviceを利用してそれぞれのPodに負荷分散しています。

**\* 自宅インフラでは、LoadBalancer Serviceを使用するためにKubernetesクラスター上に[MetalLB](https://metallb.universe.tf/)を導入しています。**

本ブログを公開するために、全てのブログ記事はNFS(Network File System)上に保管されています。
そのため、PersistentVolumeリソースとPersistentVolumeClaimリソースを利用してNFS上にあるブログ記事を保存しているディレクトリを各Podに対して公開ディレクトリとしてマウントすることで本ブログの公開を可能にしています。

&nbsp;



### techblog-synchronizing

本ブログの運営には、HUGOとGitを利用してブログ記事の作成と管理をしています。
また、ブログ記事の継続的デリバリーを実現させるために全てのブログ記事をGitHubリポジトリ上で管理しています。

このコンポーネントは、ブログ記事を管理しているGitHubリポジトリ上のmasterブランチに対する更新があったとき、ブログ記事を保管しているNFSに対して更新内容を反映させる役割を担っています。

これによって、GitHubリポジトリ上のmasterブランチへのpushをトリガーとして継続的デリバリーが行われるため、ブログの更新や内容の修正などが ***"git push"*** で行えるようになっています。

Pod上では、以下のスクリプトが動作しています。

```bash
#!/bin/bash

LOOP_STOP_FLAG=0
trap 'LOOP_STOP_FLAG=1; trap 1 2 3 4 9 15' 1 2 3 4 9 15

if [ ${GITHUB_ACCOUNT} = "" ] || [ ${GITHUB_REPOSITORY_NAME} = "" ]; then
    exit 1
fi

if [ ${DIRECTORY_NAME} = "" ]; then
    DIRECTORY_NAME=${GITHUB_REPOSITORY_NAME}
fi

rm -rf ${DIRECTORY_NAME}
git clone https://github.com/${GITHUB_ACCOUNT}/${GITHUB_REPOSITORY_NAME}.git ${DIRECTORY_NAME}

while [ $LOOP_STOP_FLAG -eq 0 ]
do
    git fetch origin

    if [ `git diff origin/master | wc -l` -ne 0 ]; then
        git pull
    fi

    sleep 1
done

exit 0
```

***GITHUB_ACCOUNT*** や ***GITHUB_REPOSITORY_NAME*** などの環境変数はCofigMapリソースを利用して設定しています。

&nbsp;



### ngrok-tunneling

このコンポーネントは、ngrokに対してHTTPトンネリング接続することで外部公開用URLを取得する役割と、外部公開用URLに対するアクセスを本ブログのwebサーバとしての役割を担っているongoingコンポーネントにプロキシしたり、ローカル上でのみアクセス可能なngrokのwebインターフェースに対して外部からのアクセスを可能するリバースプロキシサーバとしての役割を担っています。

前者の役割は ***ngrok-http*** コンテナ、後者は ***ngrok-proxy*** コンテナが担っています。

ngrokのHTTPトンネリングに必要なAuthトークンはSecretリソースから環境変数として取得しています。
また、リバースプロキシサーバにはnginxを利用しており、プロキシに関する設定にはConfigMapリソースを使用しています。


筆者はngrokのFreeプランを利用しているのですが、Freeプランでは4つのリージョンにつき1コネクションのみ可能となっています。
そのため、古い接続が残っていると新しい接続を試みても接続を確立させることがでません。
したがって、Devlopmentのアップデート戦略には ***RollingUpdate*** を指定せず ***Recreate*** を指定しています。

&nbsp;



### ngrok-http-webhook

最後に、このコンポーネントは、ngrokサービスから取得した外部公開用URLに変更があったときに指定したエンドポイントに対してwebhookする役割を担っています。


Pod上では以下のスクリプトが動作しています。

```bash
#!/bin/bash

LOOP_STOP_FLAG=0
NGROK_URL=
PREV_NGROK_URL=

trap 'LOOP_STOP_FLAG=1; trap 1 2 3 4 9 15' 1 2 3 4 9 15

if [ ${NGROK_INTERFACE:-""} = "" ] || [ ${WEBHOOK_ENDPOINT:-""} = "" ]; then
    exit 1
fi

while [ $LOOP_STOP_FLAG -eq 0 ]
do
    NGROK_URL=`curl -m 1 -s ${NGROK_INTERFACE}/api/tunnels | jq -r .tunnels[].public_url | grep --color=never https://*`

    if [ "$NGROK_URL" != "$PREV_NGROK_URL" ]; then
        STATUS_CODE=`curl -m 1 -X POST -H "Content-Type: application/json" -d "{\"ngrok_url\":\"$NGROK_URL\"}" $WEBHOOK_ENDPOINT -o /dev/null -w '%{http_code}' -s`
    fi

    PREV_NGROK_URL=$NGROK_URL

    sleep 1
done

exit 0
```

***NGROK_INTERFACE*** や ***WEBHOOK_ENDPOINT*** といった環境変数はComfigMapリソースを利用して設定しています。


&nbsp;



### 最後に

このブログはKubernetesの勉強を兼ねて作成しました。

次は何らかのWebサービスの開発を題材として、クラウドネイティブの勉強を行っていきたいと考えています。

&nbsp;
