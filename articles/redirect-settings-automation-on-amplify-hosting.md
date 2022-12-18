---
title: "Amplify Hosting × Next.js Dynamic Routingのリダイレクト設定を自動化する"
emoji: "🤖"
type: "tech"
topics: ["nextjs", "amplify"]
published: false
---

こんにちは、じゃがです👋
本記事は [AWS AmplifyとAWS×フロントエンド Advent Calendar 2022](https://qiita.com/advent-calendar/2022/amplify)、18日目の記事です

# 概要
Next.js の Dynamic Routing を使ったアプリケーションを Amplify Hosting でホストする際には、適切なリダイレクト設定が必要です

毎度ページを足すたびにリダイレクト設定を手動で書き換えるのは大変なので、ビルド時に自動でリダイレクト設定を更新しちゃいましょう！

ソースコードは以下で見ることができます

https://github.com/jaga810/amplify-hosting-redirect-setting-automation

# Dynamic Routing に必要なリダイレクト設定
具体的には下記のようなリダイレクト設定が必要です

![](/images/redirect-settings-automation-on-amplify-hosting/redirect-setting.png)

`pages` 配下にページを足す度、リダイレクト設定を追加していく必要があります
手動で更新するとどうしてもヒューマンエラーが起こってしまいますし、何より面倒です

こうした定型作業は自動化するに限りますね！


# 自動化
リダイレクト設定を更新するシェルスクリプト `update_amplify_redirect_setting.sh` を、Amplify Hosting のビルド時に `postBuild` フェーズで実行するようにします

## `update_amplify_redirect_setting.sh` 
Amplify Hosting は JSON ファイルを用いてリダイレクト設定を更新することができます
これを利用して、以下の2ステップでリダイレクト設定を更新します

1. リダイレクト設定が記された `redirect_settings.json` の生成
2. AWS CLI を用いた `redirect_settings.json` の内容の反映

1.をさらに細分化すると、以下三つの設定を行います
- `/pages` 配下のファイル群に対応した 200 リダイレクトの設定
- アプリで必要な 301 リダイレクトの設定
- 想定外のパスへのアクセスに対する 404 リダイレクトの設定


https://github.com/jaga810/amplify-hosting-redirect-setting-automation/blob/main/update_amplify_redirect_setting.sh

手元で実行する場合は以下のコマンドを実行します

```bash
chmod +x update_amplify_redirect_setting.sh
./update_amplify_redirect_setting.sh
```

AWS CLI がインストールされていない環境の場合は以下のようなエラーが出ますが、リダイレクト設定の JSON ファイル自体は出力されるので無視いただいて結構です

```bash
./update_amplify_redirect_setting.sh: line 97: /usr/local/bin/aws: No such file or directory
```


## Amplify Hosting のビルド時にスクリプトを自動実行

### `amplify.yml`
Amplify Hosting では、アプリケーションのルートディレクトリ直下に `amplify.yml` を置くことでビルド設定をカスタマイズできます
[参考](https://docs.aws.amazon.com/ja_jp/amplify/latest/userguide/build-settings.html)


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

## お金

Amplify Hosting は API をコールしただけでは課金されないので、本記事で紹介したリダイレクト設定の自動化を足すことによって追加料金が発生することはありません
詳しい課金体系は[公式ドキュメント](https://aws.amazon.com/jp/amplify/pricing)をどうぞ

## 特定のブランチでのみリダイレクト設定の更新を行う

本記事で紹介した設定をすると、[プレビュー](https://docs.aws.amazon.com/amplify/latest/userguide/pr-previews.html)や main ブランチ以外の環境へのデプロイ時にもリダイレクト設定の更新が走ります
`pages` 配下にページを追加していく場合は問題ありませんが、既存のページ構成を大きく変更したい場合などはご注意ください
dev ブランチでページ構成を変えたものがリダイレクト設定に反映されてしまい、main ブランチが紐づく本番環境で意図しないリダイレクトが起こってしまう可能性があります
(Amplify Hosting のリダイレクト設定はブランチごとに設定できるわけではないため)

main ブランチでのみリダイレクト設定を更新したい場合は、`AWS_BRANCH` 環境変数を利用すると良さそうです。
([ビルド設定の構成](https://docs.aws.amazon.com/ja_jp/amplify/latest/userguide/build-settings.html#branch-specific-build-settings))

```yml:amplify.yml
postBuild:
    commands:
        - if [ "${AWS_BRANCH}" = "main" ]; then ./update_amplify_redirect_setting.sh; fi
```

# まとめ
Amplify Hosting でビルドが走るたびに、自動的にリダイレクト設定を更新する方法を紹介しました
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
