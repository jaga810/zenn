---
title: "Amplify Hosting × Next.js Dynamic Routingのリダイレクト設定を自動化する"
emoji: "🤖"
type: "tech"
topics: ["nextjs", "amplify"]
published: false
---

こんにちは、最近はXXXXなじゃがです👋
本記事は [AWS AmplifyとAWS×フロントエンド Advent Calendar 2022](https://qiita.com/advent-calendar/2022/amplify)、18日目の記事です。

# 概要
Next.jsのDynamic Routingを使ったアプリケーションをAmplify Hostingでホストする際には、適切なリダイレクト設定が必要です

毎度ページを足すたびにリダイレクト設定を手動で書き換えるのは大変なので、ビルド時に自動でリダイレクト設定を更新しちゃいましょう！

ソースコードは以下で見ることができます

https://github.com/jaga810/amplify-hosting-redirect-setting-automation

# Dynamic Routing に必要なリダイレクト設定
Next.jsのDynamic Routingを使ったアプリケーションをAmplify Hostingでホストする際には、下記のようなリダイレクト設定が必要です。

![](/images/redirection-settings-automation-on-amplify-hosting/redirect-setting.png)

`pages` 配下にページを足すたび、この設定を変えていく必要があります。
手動で更新するとどうしてもヒューマンエラーが起こってしまいますし、何より面倒です。
私が所属する[Nstock](https://nstock.co.jp)でも、うっかり更新を忘れてしまうことがあり、自動化することに決めました。

# 自動化
ここではリダイレクト設定を更新するシェルスクリプト `update_amplify_redirect_setting.sh` を、Amplify Hostingのビルド時に `postBuild` フェーズで実行するようにします。

## `update_amplify_redirect_setting.sh` 
このファイルでは主に2つのことを実施します。

1. リダイレクト設定が記された `redirection_settings.json`の生成
    1. `/pages` 配下のファイル群の読み込みと、それに対応したリダイレクト設定の追加
    2. アプリケーションの仕様上、301リダイレクトしたいページの設定追加
    3. 想定外のパスへのアクセスに対し404ページを返す設定
2. AWS CLIを用いた `redirection_settings.json` の内容の反映


https://github.com/jaga810/amplify-hosting-redirect-setting-automation/blob/main/update_amplify_redirect_setting.sh


## `amplify.yml` でビルド時に `update_amplify_redirect_setting.sh` を実行する
Amplify Hostingでは、アプリケーションのrootディレクトリ直下に`amplify.yml`を置くことでビルド設定をカスタマイズできます。[参考](https://docs.aws.amazon.com/ja_jp/amplify/latest/userguide/build-settings.html)


https://github.com/jaga810/amplify-hosting-redirect-setting-automation/blob/main/amplify.yml

## IAMの設定
デフォルトの設定ではビルドパイプライン中で `aws amplify update-app` を実行するための権限がありませんので、足してあげましょう。
(ただし、BackendもAmplify CLIで作成してる場合は権限が付与されているはずです)

1. Generalの設定を開き、Editを押す
![](/images/redirection-settings-automation-on-amplify-hosting/general-settings.png)
2. Create new roleを押す
![](/images/redirection-settings-automation-on-amplify-hosting/create-new-role.png)
3. IAMの画面に移動するので、初期設定のまま遷移してロールを作成する(ただし、Amplifyのビルドパイプラインが利用するIAMの権限を絞りたい場合は、適切な権限を付与するようカスタマイズしてください)
4. 作成したロールを使うよう設定する
![](/images/redirection-settings-automation-on-amplify-hosting/set-new-role.png)


# おまけ

## お金

AmplifyはAPIをコールしただけでは課金されないので、この設定を足すことによって追加の料金が発生することはありません。
詳しい課金体系は[公式ドキュメント](https://aws.amazon.com/jp/amplify/pricing)をご覧ください。

## 特定のブランチでのみ行う

本記事で紹介した設定をすると、[プレビュー](https://docs.aws.amazon.com/amplify/latest/userguide/pr-previews.html)やmainブランチ以外の環境へのデプロイ時にもリダイレクト設定の更新が走ります。
pages配下にページを追加していく場合は問題ないはずですが、既存のページ構成を大きく変更したい場合などはご注意ください。
devブランチでページ構成を変えたものがリダイレクト設定に反映されてしまい、mainブランチが紐づく本番環境で意図しないリダイレクトが起こってしまう可能性があります。

mainブランチでのみリダイレクト設定を更新したい場合は、`AWS_BRANCH` 環境変数を利用すると良さそうです。
([ビルド設定の構成](https://docs.aws.amazon.com/ja_jp/amplify/latest/userguide/build-settings.html#branch-specific-build-settings))

```yml:amplify.yml
postBuild:
    commands:
        - if [ "${AWS_BRANCH}" = "main" ]; then ./update_amplify_redirect_setting.sh; fi
```

# まとめ
Amplify Hostingでビルドが走るたびに、自動的にリダイレクト設定を更新する方法を紹介しました。
`amplify.yml` と `update_amplify_redirect_setting.sh` をコピペし、IAMの設定をすればすぐに使うことができると思いますので、是非ご活用ください〜！

# 参考
この記事は以下偉大な二つの記事の肩に乗って執筆しました。
- [【Next.js x Amplify】リダイレクト設定が面倒なので一括で出力するシェルスクリプトを書きました【書き換えて、リダイレクト】 - Qiita](https://qiita.com/ItsukiN32/items/d895f32bdeeb757fb85e) 
- [Amplify HostingでのNext.jsのDynamic Routesの設定](https://zenn.dev/nus3/articles/e3da1bdb3ef302962f07)
