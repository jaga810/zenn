---
title: 'AmplifyでウェブフロントエンドのログをCloudWatchに送る'
emoji: '👀'
type: 'tech' # tech: 技術記事 / idea: アイデア
topics: ['Amplify', 'AWS', 'React', 'CloudWatch', 'Log']
published: true
---

# 概要
最近検証する機会があったので、Amplifyが導入されたウェブフロントエンド(React)からAmazon CloudWatch Logsにログを送り、閲覧するまでの手順をまとめてみました。

::: details 動作確認したバージョン
- Amplify CLI -> @aws-amplify/cli@7.6.13
- Amplify Libraries -> aws-amplify@4.3.13
:::

::: details 書かないこと、及び参考記事
- Amplifyの基礎
    - [AWS Amplify で始める、サクッとアプリ開発](https://aws.amazon.com/jp/builders-flash/202103/amplify-app-development)
    - [Getting started - React - AWS Amplify Docs](https://docs.amplify.aws/start/q/integration/react/)
- AWS IAMの概要
    - [AWS Hands-on for Beginners ハンズオンはじめの一歩: AWS アカウントの作り方 & IAM 基本のキ](https://pages.awscloud.com/event_JAPAN_Ondemand_Hands-on-for-Beginners-1st-Step_LP.html)
- ウェブフロントエンドでのログの取り回しについて
    - [フロントエンドエンジニアなら知っておきたい、JavaScriptのログ収集方法まとめ](https://www.webprofessional.jp/logging-errors-client-side-apps/)
    - [Web VitalsとJavaScriptエラーの可視化 - フロントエンドにおけるObservabilityとは](https://speakerdeck.com/watilde/visualize-web-vitals-and-javascript-error)
    - (自分も知見がないので、良いコンテンツがあったらコメントで教えていただけると嬉しいです!)
:::

# Amplify Logger とは
[Amplify Logger](https://docs.amplify.aws/lib/utilities/logger/q/platform/js/)はAWS Amplifyの機能の一つです。
ドキュメントにはローカルでエラーログを出力する仕組みしか記述されていませんが、実はAmazon CloudWatch Logsにログを送ることが可能です。


# 手順
## 0. Bootstrap
まず、Reactのプロジェクトを作成し、Amplifyを初期化します。

```sh
npx create-react-app amplify-logger-test
cd amplify-logger-test
amplify init # すべてデフォルトの選択肢でOK
```

次に、ユーザーが認証する前と後のどちらでもログを送れることを確認するために、認証基盤をセットアップします。

```sh
amplify add auth
```

最初の選択肢でManual Configurationを選びます。その後は以下のようにセットアップします。デフォルトの選択肢を選ばないほうがよいものは、__太字__ で示しています。

:::details プロンプトの受け答えの参考
> __Do you want to use the default authentication and security configuration? Manual configuration__
 Select the authentication/authorization services that you want to use: User Sign-Up, Sign-In, connected with AWS IAM controls (Enables
 per-user Storage features for images or other content, Analytics, and more)
 Provide a friendly name for your resource that will be used to label this category in the project: amplifyloggertest18699a5818699a58
 Enter a name for your identity pool. amplifyloggertest18699a58_identitypool_18699a58
 __Allow unauthenticated logins? (Provides scoped down permissions that you can control via AWS IAM) Yes__
 __Do you want to enable 3rd party authentication providers in your identity pool? No__
 Provide a name for your user pool: amplifyloggertest18699a58_userpool_18699a58
 Warning: you will not be able to edit these selections.
 How do you want users to be able to sign in? Username
 __Do you want to add User Pool Groups? No__
 __Do you want to add an admin queries API? No__
 Multifactor authentication (MFA) user login options: OFF
 Email based user registration/forgot password: Enabled (Requires per-user email entry at registration)
 Specify an email verification subject: Your verification code
 Specify an email verification message: Your verification code is {####}
 Do you want to override the default password policy for this User Pool? No
 Warning: you will not be able to edit these selections.
 What attributes are required for signing up? Email
 Specify the app's refresh token expiration period (in days): 30
 Do you want to specify the user attributes this app can read and write? No
 Do you want to enable any of the following capabilities?
 __Do you want to use an OAuth flow? No__
 __Do you want to configure Lambda Triggers for Cognito? No__
:::

ポイントは

> Allow unauthenticated logins? (Provides scoped down permissions that you can control via AWS IAM)

をYesにしているところです。
未認証のユーザーがアプリを利用しているときにもログを送ることができるように、未認証ユーザーのログインを許可しています。一方で、未認証ユーザーのログは必要ない、ということであれば、許可しないほうが良いでしょう。



## 1. ウェブフロントエンドからCloudWatch Logsにログを送るための権限設定
`$ amplify init`するとデフォルトで作られるリソースの中に、__Auth Role__ と __Unauth Role__ があります。この２つのIAMロールは、未認証ユーザーや認証済みユーザーへ、AWSのサービスを直接呼び出す権限を与えるために利用されます。

- Unauth Role : 未認証ユーザーへ権限を付与する
- Auth Role : 認証済みユーザーへ権限を付与する

たとえば`$ amplify add storage`でS3バケットを追加した場合、S3のサービスAPIをウェブフロントエンドから直接呼び出して画像や動画をアップロード・閲覧することを許可できます。
この際、権限付与にはUnauthRoleやAuthRoleが用いられ、Amplify CLIはS3バケットに読み書きする権限をそれぞれのIAMロールに付与します。

現状Amplify CLIを使用してAuthRoleとUnauthRoleにCloudWatch Logsへ書き込む権限を与えることがでないため(`$ amplify add logger`のようなコマンドがない)、手動で権限を付与する必要があります。

本記事ではAmplfiyが作成したリソースをカスタマイズすることができる、[Amplify Override](https://aws.amazon.com/jp/blogs/news/override-amplify-generated-backend-resources-using-cdk/)を用いて権限を付与します。

```sh
amplify override project
```

作成された`override.ts`ファイルを開き、以下のように書き換えます。

```typescript:amplify/backend/awscloudformation/override.ts
import { AmplifyRootStackTemplate } from '@aws-amplify/cli-extensibility-helper';

export function override(resources: AmplifyRootStackTemplate) {
    const accountId = 'Your Account ID' // 自身のAWSアカウントID
    const region = 'us-east-1' // ログを送りたいリージョン
    const loggerPrefix = 'amplify-logger' // ロググループのPrefix(後述)

    const loggerPolicy =
    {
        'policyDocument': {
            'Version': '2012-10-17',
            'Statement': [
                {
                    'Effect': 'Allow',
                    'Action': 'logs:PutLogEvents',
                    'Resource': `arn:aws:logs:${region}:${accountId}:log-group:/${loggerPrefix}/*:log-stream:*`,
                },
                {
                    'Effect': 'Allow',
                    'Action': [
                        'logs:CreateLogStream',
                        'logs:CreateLogGroup',
                        'logs:DescribeLogStreams',
                    ],
                    'Resource': `arn:aws:logs:${region}:${accountId}:log-group:/${loggerPrefix}/*`
                },
                {
                    'Effect': 'Allow',
                    'Action': [
                        'logs:DescribeLogGroups',
                    ],
                    'Resource': `arn:aws:logs:${region}:${accountId}:log-group:*`
                }
            ]
        },
        'policyName': 'amplifyLoggerCWLogsPolicy'
    }

    resources.authRole.policies = [
        loggerPolicy
    ]
    resources.unauthRole.policies = [
        loggerPolicy
    ]
}
```

:::details 記述内容をより理解するためのヒント
1. アプリのユーザーが任意のログストリームにログを出力できないように、`logs:PutLogEvents`を実行できるリソースは`loggerPrefix`がついたログストリームに絞られるようにしています
2. `logs:DescribeLogGroups`は任意のロググループを参照する必要があるため、`log-group:*`としています
3. projectのoverrideを用いて任意のIAMポリシーを追加することができます。このとき、Storageカテゴリなど、他のカテゴリによって付与されたIAMポリシーは消えません
:::

CloudWatch Logsではログストリームとロググループという単位でログを集積します。[ドキュメント](https://docs.aws.amazon.com/ja_jp/AmazonCloudWatch/latest/logs/Working-with-log-groups-and-streams.html)によれば

> ログストリームは、同じソースを共有する一連のログイベントです。CloudWatch Logs でのログの各ソースで各ログストリームが構成されます。
ロググループは、保持、監視、アクセス制御について同じ設定を共有するログストリームのグループです。ロググループを定義して、各グループに入れるストリームを指定することができます。1 つのロググループに属することができるログストリームの数に制限はありません。

ログの設計方針は様々考えられますが、本記事では[CloudwatchLogsのロググループ設計には気をつけよう | DevelopersIO](https://dev.classmethod.jp/articles/design-cloudwatchlogs-log-group/)を参考として、以下のようにしました。

- ロググループ : アプリケーションの環境ごとにユニーク (ex. 本番環境のロググループと開発環境のロググループは分かれている)
- ログストリーム : クライアントごとにユニーク (ex. ユーザーAのログとユーザーBのログは異なるログストリームに格納される)

特にログストリームは`1 秒、1 ログストリームあたり 5 リクエスト`の制限があるため([ドキュメント](https://docs.aws.amazon.com/ja_jp/AmazonCloudWatch/latest/logs/cloudwatch_limits_cwl.html))、クライアントごとにユニークに分けたほうが良いと思います。

以上でバックエンドの設定が終わったため、変更をクラウド上に反映します。

```sh
amplify push -y
```

## 2. ウェブフロントエンドでログを送る設定
ReactアプリケーションからCloudWatch Logsに送る設定をしていきます。
まず、必要なライブラリをインストールしましょう。

```sh
npm install aws-amplify @aws-amplify/ui-react
```

次に、`App.js`を開き、以下のように書き換えます。

```javascript:src/App.js
import { Amplify, Logger, AWSCloudWatchProvider} from 'aws-amplify'

import { Authenticator } from '@aws-amplify/ui-react'
import '@aws-amplify/ui-react/styles.css'

import awsExports from './aws-exports'

const loggerPrefix = 'amplify-logger'
const appName = 'test-app'
const username = 'test-user' //実際に利用するときは、未認証ユーザーにはUUID、認証済みユーザーにはusernameなどの識別子を入れてログを取ると良さそう

Amplify.configure({
  Logging: {
    logGroupName: `/${loggerPrefix}/${appName}/${process.env.NODE_ENV}`,
    logStreamName: username,
  },
  ...awsExports,
})

const LOG_LEVEL = 'INFO' //どのレベルのログまでロギングするか

const logger = new Logger('TestLogger', LOG_LEVEL)
Amplify.register(logger)
logger.addPluggable(new AWSCloudWatchProvider())

export default function App() {
  logger.error(username, 'test error')
  logger.warn(username, 'test warn')
  logger.info(username, 'test info')
  logger.debug(username, 'test debug')

  return (
    <Authenticator>
      {({ signOut, user }) => (
        <main>
          <h1>Hello {user.username}</h1>
          <button onClick={signOut}>Sign out</button>
        </main>
      )}
    </Authenticator>
  );
}
```

:::details 記述内容をより理解するためのヒント
1. `LOG_LEVEL`ではどのレベルのログまでログングするか指定することができます
    - `VERBOSE` < `DEBUG` < `INFO` < `WARN` < `ERROR`
    - デフォルトの値は`WARN`のため、指定しなければ`WARN`と`ERROR`レベルのログが出力されます
    - LOG_LEVELの詳しい指定方法について知りたい人は[こちらのドキュメント](https://docs.amplify.aws/lib/utilities/logger/q/platform/js/#setting-logging-levels)をご参照下さい
2. CloudWatch Logsの連携に必要な記述部分のソースは以下のPull Requestです
    - [Core/cloudwatch logging by eddiekeller · Pull Request #8309 · aws-amplify/amplify-js](https://github.com/aws-amplify/amplify-js/pull/8309)
3. Auth RoleとUnauth Roleへ与えた権限と、おなじPrefixをもつロググループを指定しています。
:::

## 3. 動作確認

では、実際にアプリを動かしてログがCloudWatch Logsに出力されることを確かめてみましょう。

```sh
npm start
```

検証ツールを使うと、以下のようにログが出ていることが確認できます。

![](/images/amplify-logger-with-cwlogs/app-ss-unauthenticate.png)

次に、CloudWatch Logsの画面を確認してみましょう
1. [ロググループの画面](https://console.aws.amazon.com/cloudwatch/home#logsV2:log-groups)を開く
2. フィルターに`/amplify-logger/`と打ち込み、でてきた`/amplify-logger/test-app/development`ロググループをクリックします
![](/images/amplify-logger-with-cwlogs/cwlogs-log-groups.png)
3. ログストリームの中から、`test-user`をクリックします
![](/images/amplify-logger-with-cwlogs/cwlogs-log-streams.png)
4. フロントエンドから送ったログが格納されていることを確認します
![](/images/amplify-logger-with-cwlogs/cwlogs-log-events.png)

同様の手順で、認証後にもログが送れることを確認できますので、是非試してみて下さい。

# まとめ
Amplify Loggerを利用してCloudWatch Logsにログを送る方法をまとめてみました。現状、自前でAuth RoleとUnauth Roleのポリシーを追加しなければならないのが少し手間ですが、セットアップする際の参考になれば幸いです。
感想や疑問点あれば気軽にコメント下さい！

# 謝辞的な
[Amplify Advent Calndar 2021](https://qiita.com/advent-calendar/2021/amplify)の[AWS AmplifyでCloudWatchLogsにカジュアルにログを送りたい](https://qiita.com/jacoyutorius/items/3cbed0075125905345a9)をベースとして、いくつか補足を加えてみました。
執筆者の[@jacoyutorius](https://twitter.com/jacoyutorius)さん、Amplify LoggerとCloudWatch Logsの連携を教えていただいた[@enish](https://zenn.dev/enish)さん、ありがとうございます!