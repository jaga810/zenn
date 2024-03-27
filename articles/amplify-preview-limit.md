---
title: '突然Amplify Hostingのプレビュー環境が作成されなくなったあなたへ'
emoji: '🥺'
type: 'tech' # tech: 技術記事 / idea: アイデア
topics: ['amplify']
published: true
publication_name: "nstock"
---

こんにちは！Nstock でエンジニアをしています、jagaです 🥔

NstockではNext.jsのホスティングに[Amplify Hosting](https://docs.aws.amazon.com/ja_jp/amplify/latest/userguide/welcome.html)を使っています。
特にAmplify Hostingの[プレビュー機能](https://docs.aws.amazon.com/ja_jp/amplify/latest/userguide/pr-previews.html)は、プルリクエストの変更差分を確認するのに重宝しています。
ですがある時、いきなりプレビュー環境が作成されなくなってしまいました。

あれこれ調べて原因がわかったので、原因と対処法について記します。

# 原因

アプリケーションのブランチ数のクォータに達したことが原因でした。


![Quota](/images/amplify-preview-limit/quota.png)
*https://docs.aws.amazon.com/ja_jp/amplify/latest/userguide/welcome.html*

Amplifyで1アプリケーションに紐づけられるブランチ数は50が上限で、引き上げはできません。
プレビュー環境もこのクォータを消費します。普段の開発でこのクォータを意識することはほとんどないと思います。

問題はプレビュー環境はプルリクエストをマージしても環境が消えない不具合が時折あることです。
Nstockの場合、以下の二点が重なり、クォータに達してしまいました。

- 不具合によって不要なプレビュー環境が多数残ってしまっていた
- 直近Renovateが作成するPR数の上限を引き上げたことで、普段より多くのプレビュー環境が立ち上がってしまった

ちなみに今回は、プレビュー環境が自動で立ち上がらないので、やむをえず手動でブランチを選択して環境を立てようとした時にエラーがでて気づきました。

![Error](/images/amplify-preview-limit/error.png)


# 対処法
不要なプレビュー環境を削除することで問題は修正できます。
Amplifyのコンソール上からプレビュー環境を消すことはできないため、AWS CLIから削除する必要があります。
以下のissueにある通りコマンドを叩いて、プレビュー環境を消しましょう。

```shell
# list branches for app
aws amplify list-branches --app-id <amplify_app_id>

# delete branch for app
aws amplify delete-branch --app-id <amplify_app_id> --branch-name <your_branch_name>

# example
aws amplify delete-branch --app-id <amplify_app_id> --branch-name pr-1000
```
*https://github.com/aws-amplify/amplify-hosting/issues/472*

# 最後に
「最近有効化したGitHub Enterpriseが原因なんだろうか...」
「特定の人が作成したプルリクエストにはプレビュー環境が生成されているように見えるぞ？」
「AmplifyのGitHubアプリをアップデートしていないからかな」

など、同僚の[matamatanot](https://twitter.com/matamatanot)さんと試行錯誤した結果でした。
(一番苦労する、不要なプレビュー環境を消す作業は[matamatanot](https://twitter.com/matamatanot)さんがやってくださったので感謝の意しかない)

この記事を読んだ方がスッと問題を解決できますように！

