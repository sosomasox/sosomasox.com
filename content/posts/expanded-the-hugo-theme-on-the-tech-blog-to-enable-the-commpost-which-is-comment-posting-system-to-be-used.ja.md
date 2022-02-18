---
title: "TechブログのHUGOテーマを拡張してコメント投稿システムである『COMMPOST』を利用できるようにした話"
date: 2020-08-08T00:10:36+09:00
Description: ""
thumbnail: /images/expanded-the-hugo-theme-on-the-tech-blog-to-enable-the-commpost-which-is-comment-posting-system-to-be-used/thumbnail.png
Tags: []
Categories: []
DisableComments: false
---

&nbsp;

[前回の記事](https://techblog.sosomasox.com/posts/commpost-a-comment-posting-system/)では、このTechブログにコメント機能を追加するために開発したコメント投稿システム『COMMPOST』を紹介しました。

本稿では、HUGOのテーマを拡張して『COMMPOST』を利用できるようにしましたのでそのお話をさせて頂きます。

本ブログではHUGOのテーマである [**anatole**](https://github.com/lxndrblz/anatole) を拡張して利用しています。

今まではTwitterのOGPに対応させたり、レイアウトを変更したりしました。

今回はコメント投稿システム『COMMPOST』を利用できるに拡張していきます。


&nbsp;

まず、**layouts/partialsディレクトリ** 下に『COMMPOST』のフロントエンドを表示する下記のHTMLファイルを配置します。

&nbsp;

```:layouts/partials/commpost.html
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
```


&nbsp;

次に **layouts/\_default/single.html** ファイルを編集します。

適当なところに下記を追加します。

下記のように設定することで、HUGOの設定ファイルに対して **_params_** セクションに **_"commpost"_** というパラメータに **_"true"_** と設定することで
コメント投稿システム『COMMPOST』を利用できるようなります。


&nbsp;

```:layouts/_default/single.html
{{ if eq .Site.Params.commpost true }}
{{ partial "commpost.html" . }}
{{ end }}
```

&nbsp;

最後にHUGOの設定ファイルである **config.toml** を編集します。

**_params_** セクション下に **_"commpost = true"_** というパラメータを追加します。

&nbsp;

```cofig.toml

[params]
title = "on-going's Tech Blog"
author = "techblog.on-going."
profilePicture = "images/profile.jpg"
thumbnail = "images/profile.jpg"
twitter = "izewfktvy533"
commpost = true

```

&nbsp;

これでいつものようにhugoコマンドを実行することですべてのブログ記事でコメント投稿システム『COMMPOST』が使えるようになりました。

&nbsp;
