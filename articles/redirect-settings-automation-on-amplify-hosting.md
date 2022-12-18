---
title: "Amplify Hosting × Next.js Dynamic Routesのリダイレクト設定を自動化する"
emoji: "🤖"
type: "tech"
topics: ["nextjs", "amplify"]
published: false
---

こんにちは、じゃがです 👋
本記事は [AWS AmplifyとAWS×フロントエンド Advent Calendar 2022](https://qiita.com/advent-calendar/2022/amplify)、18日目の記事です

# 概要
Next.js の Dynamic Routes を使ったアプリケーションを Amplify Hosting でホストする際には、適切なリダイレクト設定が必要です

毎度ページを足すたびにリダイレクト設定を手動で書き換えるのは大変なので、ビルド時に自動でリダイレクト設定を更新しちゃいましょう！

ソースコードは以下で見ることができます

https://github.com/jaga810/amplify-hosting-redirect-setting-automation

# Dynamic Routes に必要なリダイレクト設定

## リダイレクト設定を行わない場合どうなるか？
以下では [jaga810/amplify-hosting-redirect-setting-automation](https://github.com/jaga810/amplify-hosting-redirect-setting-automation) のルートディレクトリで作業しているものとします

```bash
$ tree pages
pages
├── _app.tsx
├── _document.tsx
├── index.tsx
├── posts
│   ├── [id]
│   │   └── index.tsx
│   └── index.tsx
└── settings
    ├── private
    │   └── index.tsx
    └── public
        └── index.tsx
```

`pages/[id]/index.tsx` の部分が Dynamic Routes を利用している部分になります

ビルドを実行します

```bash
npm run build # next build && next export
```

out ディレクトリに以下のようなファイル群が生成されます

```bash
$ tree out -P *.html -I _next
out
├── 404.html
├── index.html
├── posts
│   └── [id].html
├── posts.html
└── settings
    ├── private.html
    └── public.html
```

これを Amplify Hosting でホストし、`/posts/123` のようなパスにアクセスすると、以下のように `/404.html` にリダイレクトされてしまいます

![404にリダイレクトされる様子](/images/redirect-settings-automation-on-amplify-hosting/404.png)

`/posts/123` にアクセスを試みても、実際にホストされているファイルは `posts/[id].html` であって `posts/123.html` ではありません

そのため、リクエストパスに該当するファイルが見つからず、404 ページにリダイレクトされてしまうのです

この問題を回避するためにはどうしたら良いのでしょうか？

## Dynamic Routes のためのリダイレクト設定
上述の問題を回避するためには、下記のようなリダイレクト設定が必要です

![リダイレクト設定の例](/images/redirect-settings-automation-on-amplify-hosting/redirect-setting.png)

赤い枠線で囲まれた箇所は、 `/posts/<id>` (`<id>`は任意の文字列) にマッチするリクエストを、`/posts/[id]` にリダイレクトする、という設定です

たとえば `/posts/123` へのリクエストが `/posts/<id>` にマッチしてリダイレクトされ、 `/posts/[id].html` がユーザーに返されることになります

ちなみに、Amplify Hosting のリダイレクト設定において、`<id>` はプレイスホルダーと呼ばれています

![リダイレクト設定におけるプレイスホルダー](/images/redirect-settings-automation-on-amplify-hosting/redirect-placeholder.png)
[リダイレクトを使用する - AWS Amplifyホストする](https://docs.aws.amazon.com/ja_jp/amplify/latest/userguide/redirects.html#placeholders)

今回の用途のように、特定のパス構造でファイルが見つからない場合にリダイレクトをかけたい場合にプレイスホルダーが重宝します

また、プレイスホルダーは変数のように扱うことができ、リダイレクト先を動的に指定することもできます


## 毎回手作業は辛いよ...

このように、`pages` 配下に Dynamic Routes を利用するページを足す度、Amplify Hosting のリダイレクト設定を追加していく必要があります

手動で更新するとどうしてもヒューマンエラーが起こってしまいますし、何より面倒です

定型作業は自動化しちゃいましょう！


# 自動化
リダイレクト設定を更新するシェルスクリプト `update_amplify_redirect_setting.sh` を、Amplify Hosting のビルド時に `postBuild` フェーズで実行するようにします

## `update_amplify_redirect_setting.sh` 
Amplify Hosting は JSON ファイルを用いてリダイレクト設定を更新することができます

これを利用して、以下の2ステップでリダイレクト設定を更新します

1. リダイレクト設定が記された `redirect_settings.json` の生成
2. AWS CLI を用いた `redirect_settings.json` の内容の反映

https://github.com/jaga810/amplify-hosting-redirect-setting-automation/blob/main/update_amplify_redirect_setting.sh

ちなみに、1.を詳しく見ると、以下3つの設定をおこなっています
- `/pages` 配下のファイル群に対応した 200 リダイレクトの設定
- アプリで必要な 301 リダイレクトの設定
- 想定外のパスへのアクセスに対する 404 リダイレクトの設定


手元で出力されるリダイレクト設定を確認したい場合は、以下のコマンドを実行してください

```bash
chmod +x update_amplify_redirect_setting.sh 
./update_amplify_redirect_setting.sh
```

AWS CLI がインストールされていない環境の場合は以下のようなエラーが出ますが、リダイレクト設定の JSON ファイル自体は出力されるので無視いただいて大丈夫です

```bash
./update_amplify_redirect_setting.sh: line 97: /usr/local/bin/aws: No such file or directory
```


## Amplify Hosting のビルド時にスクリプトを自動実行

### `amplify.yml`
Amplify Hosting では、アプリケーションのルートディレクトリ直下に `amplify.yml` を置くことでビルド設定をカスタマイズできます
[ビルド設定の構成 - AWS Amplifyホストする](https://docs.aws.amazon.com/ja_jp/amplify/latest/userguide/build-settings.html)


https://github.com/jaga810/amplify-hosting-redirect-setting-automation/blob/main/amplify.yml#L11-L13

### IAMの設定
デフォルトの設定ではビルドパイプライン中で `aws amplify update-app` を実行する権限がありませんので、足してあげましょう

1. General 開き、Edit を押す
![Generalの設定を開き、Editを押す](/images/redirect-settings-automation-on-amplify-hosting/general-settings.png)
2. Create new role を押す
![Create new role を押す](/images/redirect-settings-automation-on-amplify-hosting/create-new-role.png)
3. IAMの画面に移動するので、初期設定のまま遷移してロールを作成する(ただし、Amplifyのビルドパイプラインが利用するIAMの権限を絞りたい場合は、適切な権限を付与するようカスタマイズしてください)
4. 作成したロールを使うよう設定する
![作成したロールを使うよう設定する](/images/redirect-settings-automation-on-amplify-hosting/set-new-role.png)


# おまけ

## コスト

Amplify Hosting は API をコールしただけでは課金されないため、本記事で紹介したリダイレクト設定の自動化を足すことによる追加料金はありません
[AWS Amplify の料金 | ウェブとモバイルのフロントエンド | Amazon Web Services](https://aws.amazon.com/jp/amplify/pricing/)

## 特定のブランチでのみリダイレクト設定の更新を行う

本記事で紹介した設定をすると、[Pull Request プレビュー](https://docs.aws.amazon.com/amplify/latest/userguide/pr-previews.html)や main ブランチ以外の環境へのデプロイ時にもリダイレクト設定の更新が走ります

`pages` 配下にページを追加していく場合は問題ありませんが、既存のページ構成を大きく変更したい場合などはご注意ください

dev ブランチでページ構成を変えたものがリダイレクト設定に反映されてしまい、main ブランチが紐づく本番環境で意図しないリダイレクトが起こってしまう可能性があります
(Amplify Hosting のリダイレクト設定はブランチごとに設定できるわけではないため)

main ブランチでのみリダイレクト設定を更新したい場合は、`AWS_BRANCH` 環境変数を利用すると良さそうです
[ビルド設定の構成 - AWS Amplifyホストする](https://docs.aws.amazon.com/ja_jp/amplify/latest/userguide/build-settings.html#branch-specific-build-settings)

```yml:amplify.yml
postBuild:
    commands:
        - if [ "${AWS_BRANCH}" = "main" ]; then ./update_amplify_redirect_setting.sh; fi
```

# まとめ
Amplify Hosting でビルドが走るたびに、自動的にリダイレクト設定を更新する方法を紹介しました 🚀

`amplify.yml` と `update_amplify_redirect_setting.sh` をコピペし、IAM の設定をすればすぐ使うことができると思いますので、是非ご活用ください〜！

# 参考
この記事は偉大な先人の肩に乗って執筆しました
- [【Next.js x Amplify】リダイレクト設定が面倒なので一括で出力するシェルスクリプトを書きました【書き換えて、リダイレクト】 - Qiita](https://qiita.com/ItsukiN32/items/d895f32bdeeb757fb85e) 
- [Amplify HostingでのNext.jsのDynamic Routesの設定](https://zenn.dev/nus3/articles/e3da1bdb3ef302962f07)

# 最後に
私が所属する [Nstock](https://nstock.co.jp) では、ソフトウェアエンジニアを募集中です🔥

https://open.talentio.com/r/1/c/nstock/pages/70402

ちょっと興味あるかも、という方はお気軽にご応募 or Twitter DM ください〜！

https://twitter.com/jagaimogmog
