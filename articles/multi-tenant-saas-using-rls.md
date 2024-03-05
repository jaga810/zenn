---
title: "RLSではじめるマルチテナントSaaS"
emoji: "🤖"
type: "tech"
topics: ["postgresql", "rls",  "java", "springboot"]
published: false
publication_name: "nstock"
---

こんにちは！Nstockのじゃがです。

NstockではマルチテナントSaaSを開発しており、テナント間のデータ分離にRow-Level Security（RLS）を利用しています。本記事ではRLSの基本から、Nstockでの利用イメージまで、SQL文やアプリケーションコードを交えて解説します。

:::message
本記事は 2023/8/4 に、別サイトで公開した記事を Zenn に移管した記事となります
:::

### 備考

- アプリケーションの実装イメージはSpring Bootですが、多くのフレームワークに存在する機能を利用しています
- PostgreSQLのRLSについて話しています

## マルチテナントアーキテクチャとRLS

Nstockは初期フェーズであり、人的リソースや金銭的リソースに余裕がありません。テナントごとに異なるDBサーバーやスキーマを用意するアーキテクチャは、リソース的に厳しいです。そのため、複数のテナントでDBサーバーを共有しつつ、 `tenant_id` カラムを用いてテナント間のデータを分離することにしました。しかし、毎回手動でWHERE句にテナント分離用の条件を追加するのは作業負荷が大きいですし、指定し忘れた場合のデータ漏洩も怖いです。

```sql
// テーブル定義
CREATE TABLE post(
    id SERIAL PRIMARY KEY
    ,content TEXT NOT NULL
    ,tenant_id UUID NOT NULL // テナント分離用のカラム
);

// データアクセス
SELECT * FROM post WHERE tenant_id = 'c0676bd5-0ca6-4e63-b9a2-522f645d8bae';
```

WHERE句を自動的に追加する方法として、ウェブフレームワークの機能や、DBのRow-Level Security（RLS）が考えられます。Nstockではより低いレイヤーでこの問題を解決するのが望ましいと考え、RLSを採用しました。

RLSは、DBテーブルの各行に対するアクセスを制御するための機能で、PostgreSQLなどいくつかのDBで利用できます。RLSを使用すると、あるテーブルの行に対して特定のユーザーのみがアクセスできる制約を設けることができます。

## RLSをハンズオン形式で学ぶ

以下ではPostgreSQL14.5を利用してRLSの利用イメージを膨らませていきます。ちなみにMySQLでは今のところサポートされていません。

:::message
RLSはPostgreSQL9.5から導入されました
:::

:::message
正確にはPostgreSQLのRLSはRow Security Policyという機能名ですが、本記事ではより一般的な呼称であるRLSを用います。[^1]
:::

### RLSポリシーの書き方

RLSを有効化するには以下の構文を利用します。

```sql
CREATE POLICY name ON table_name
    [ FOR (command) ]
    [ TO role_name [, ...] ]
    [ USING ( using_expression ) ]
    [ WITH CHECK ( check_expression ) ]
```

[https://www.postgresql.org/docs/current/sql-createpolicy.htm](https://www.postgresql.org/docs/current/sql-createpolicy.htm)[l](https://www.postgresql.jp/docs/9.5/sql-createpolicy.html)

- `name` : RLSポリシーの名前です。テーブル毎にユニークである必要があります。
- `table_name` : RLSポリシーを適用する対象のテーブル名です。
- `command` : RLSポリシーが適用されるコマンドです。デフォルトは `ALL` で、全てのコマンドに適用されます。
- `role_name` : RLSポリシーが適用されるロールです。 デフォルトは`PUBLIC`で、すべてのロールに対してポリシーが適用されます。
- `using_expression` : `SELECT`、 `UPDATE`、 `DELETE`の際にどの行が見えるか(可視性)をSQL条件式で指定します。
- `check_expression` :  テーブルに追加・更新される行に対し、 可視性の条件(`USING`句)とは異なるポリシーを使用したい場合に利用します。 `INSERT`、 `UPDATE`されたあとのレコードがこの条件式を満たすことを保証します。

まとめるとRLSポリシーは大きく以下の２つのパートに別れます。

1. RLSポリシーを適用する問い合わせの条件 (`table_name` `command` `role_name` )
2. 1.を満たす問い合わせに自動で追加する条件式 (`using_expression` `check_expression`)

1.を満たすDB問い合わせに 2.の条件式が自動で追加される、と考えるとわかりやすいです。

### RLSポリシーでマルチテナントを分離する

RLSでマルチテナント分離を行う場合、USING句に分離する条件式を書くことになります。私の調査した範囲では、主に以下の２つの変数が条件式が利用されていました。

1. ロール名
2. 実行時パラメータ

以下ではこの２つの方法を比較します。

#### 条件式のパターン 1. ロール名

DBロール名を参照して条件式を書きます。まずはテーブルを用意します。

```sql
// RLSの対象となるテーブル
CREATE TABLE post(
    id SERIAL PRIMARY KEY
    ,content TEXT NOT NULL
    ,tenant_id VARCHAR(100) NOT NULL
);

// アプリケーションが使うDBロール
CREATE ROLE app_user;
CREATE ROLE tenant_a;

// DBロールにテーブルへのアクセス権限をGRANT
GRANT SELECT, INSERT, UPDATE, DELETE, REFERENCES ON post TO app_user;
GRANT SELECT, INSERT, UPDATE, DELETE, REFERENCES ON post TO tenant_a;

// データの流し込み
INSERT INTO post (content, tenant_id) VALUES
('tenant_aの投稿', 'tenant_a'),
('tenant_bの投稿', 'tenant_b');

// 動作確認
SET ROLE app_user;
SELECT * FROM post;

id |    content     | tenant_id
----+----------------+-----------
  1 | tenant_aの投稿 | tenant_a
  2 | tenant_bの投稿 | tenant_b
(2 rows)
```

現状はRLSを設定していないため、すべてのレコードを見ることが可能です。

つぎに `current_user` 、すなわちDBに問い合わせを行うDBロールの名前と、 `tenant_id` が一致する場合のみ閲覧や操作ができる、というRLSポリシーを追加します。

```sql
// ロールの切り替え(app_user、tenant_a 以外の適切な権限をもつロールであれば何でも良い)
SET ROLE admin;

// テーブルに対しRLSを有効化する
ALTER TABLE post ENABLE ROW LEVEL SECURITY;

// ロール名とtenant_idが一致するときのみ閲覧可能なポリシーを設定する
CREATE POLICY tenant_isolation_policy ON post
    USING (tenant_id = current_user);

// ポリシーの確認
\d post;

Table "public.post"
  Column   |          Type          | Collation | Nullable |             Default
-----------+------------------------+-----------+----------+----------------------------------
 id        | integer                |           | not null | nextval('post_id_seq'::regclass)
 content   | text                   |           | not null |
 tenant_id | character varying(100) |           | not null |
Indexes:
    "post_pkey" PRIMARY KEY, btree (id)
Policies:
    POLICY "tenant_isolation_policy"
      USING (((tenant_id)::text = CURRENT_USER))
```

`Policies` セクションに、追加した `tenant_isolation_policy` の存在が確認できます。

`app_user` ロールを使って `post` テーブルを参照してみましょう。

```sql
// ロールの切り替え
SET ROLE app_user;

SELECT * FROM post;

id | content | tenant_id
----+---------+-----------
(0 rows)
```

`tenant_id` が`app_user` となるレコードはないため、0件のレコードが返ります。

次に `tenant_a` ロールを使ってアクセスしてみます。

```sql
// ロールの切り替え
SET ROLE tenant_a;

SELECT * FROM post;

id |    content     | tenant_id
----+----------------+-----------
  1 | tenant_aの投稿 | tenant_a
(1 row)
```

`tenant_id` が `tenant_a` である1件のレコードが返ります。このように、ロール名を使ったRLSポリシーでマルチテナントのデータ分離が可能です。

しかし、この方法でマルチテナントのデータを分離する場合、テナントの数だけ DBロールが必要になってしまいます。プロダクトがスケールするにつれ、DBロールの管理が複雑になるのは目に見えており、それは避けたいです。

また、ロール名含むSQLの識別子には英数字、アンダースコアしか利用できず[^2]、UUIDフォーマットは利用できません。Nstockではセキュリティ観点からIDとなるカラムにはUUID型を採用しているため、この点でもフィットしませんでした。

#### 条件式のパターン 2. 実行時パラメータ

もう一つの方法は実行時パラメータを参照する方法です。まずはテーブルを用意します。

```sql
// ロールの切り替え(app_user、tenant_a 以外の適切な権限をもつロールであれば何でも良い)
SET ROLE admin;

// RLSの対象となるテーブルを新規作成
CREATE TABLE member(
    id SERIAL PRIMARY KEY
    ,name VARCHAR(100) NOT NULL
    ,tenant_id UUID NOT NULL
);

// DBロールにテーブルへのアクセス権限をGRANT
GRANT SELECT, INSERT, UPDATE, DELETE, REFERENCES ON member TO app_user;

// データの流し込み
INSERT INTO member(name, tenant_id) VALUES
('じゃが', 'e102df93-78d3-4341-a24b-fd1a4fad6dc2'),
('いも', '258761aa-c956-4967-b9d9-4ee9c63c3603');

// 確認
SET ROLE app_user;
SELECT * FROM member;

id |  name  |              tenant_id
----+--------+--------------------------------------
  1 | じゃが | e102df93-78d3-4341-a24b-fd1a4fad6dc2
  2 | いも   | 258761aa-c956-4967-b9d9-4ee9c63c3603
(2 rows)
```

現状はRLSポリシーを設定していないため、すべてのレコードを見ることが可能です。

つぎに、実行時パラメータを利用したRLSポリシーを作成します。

```sql
// ロールの切り替え(app_user、tenant_a 以外の適切な権限をもつロールであれば何でも良い)
SET ROLE admin;

// テーブルに対しRLSを有効化する
ALTER TABLE member ENABLE ROW LEVEL SECURITY;

// 実行時パラメータを利用したRLSポリシーを追加する
CREATE POLICY tenant_isolation_policy ON member TO app_user
    USING (tenant_id = current_setting('app.current_tenant_id')::UUID);

// ポリシーの確認
\d member;

Table "public.member"
  Column   |          Type          | Collation | Nullable |              Default
-----------+------------------------+-----------+----------+------------------------------------
 id        | integer                |           | not null | nextval('member_id_seq'::regclass)
 name      | character varying(100) |           | not null |
 tenant_id | uuid                   |           | not null |
Indexes:
    "member_pkey" PRIMARY KEY, btree (id)
Policies:
    POLICY "tenant_isolation_policy"
      TO app_user
      USING ((tenant_id = (current_setting('app.current_tenant_id'::text))::uuid))
```

- `current_setting` を使い実行時パラメータを取得し、USING句の条件式に利用します
- `member` テーブルの `tenant_id`フィールドは `UUID` 型なので、 `TEXT` 型である実行時パラメータの値 `current_setting('app.current_tenant_id)` を`::UUID` を使ってキャストしています。

実際にこのポリシーを使用してみましょう。

```sql
// RLSポリシーを設定したロールに切り替え
SET ROLE app_user;

// 実行時パラメータをセットせずにクエリするとエラーが出る
SELECT * FROM member;
ERROR:  unrecognized configuration parameter "app.current_tenant_id"

// 実行時パラメータのセット
SET app.current_tenant_id = 'e102df93-78d3-4341-a24b-fd1a4fad6dc2';

// app.current_tenant_idとtenant_idと合致する1件のレコードが返る
SELECT * FROM member;

id |  name  |              tenant_id
----+--------+--------------------------------------
  1 | じゃが | e102df93-78d3-4341-a24b-fd1a4fad6dc2
(1 row)

// 別の実行時パラメータのセット
SET app.current_tenant_id = '258761aa-c956-4967-b9d9-4ee9c63c3603';

// 先程と異なる１件のレコードが返る
SELECT * FROM member;

id | name |              tenant_id
----+------+--------------------------------------
  2 | いも | 258761aa-c956-4967-b9d9-4ee9c63c3603
```

実行時パラメータは `SET` が発行されたDBセッション内でしか利用されず、他のセッションからは参照されません。セッションが閉じた際に実行時パラメータも破棄されます。

#### Nstockではどちらを採用したか

パターン2.の実行時パラメータを利用した方式では、パターン1.でみたロール名を利用したRLSポリシーと異なり、DBロールの管理が必要ありません。そのため、Nstockでは実行時パラメータを利用したRLSポリシーを採用しています。

ちなみに、RLSポリシーの名前は**テーブル毎に**一意であれば良いため、以下のRLSポリシーは併存できます。テナント分割用のRLSポリシーのように、同じ役割を持つポリシーは同じ名前にすることで、あとから見返したときに役割がわかりやすくなります。

```sql
CREATE POLICY tenant_isolation_policy ON post TO app_user
    USING (tenant_id = current_setting('app.current_tenant_id')::UUID);

CREATE POLICY tenant_isolation_policy ON member TO app_user
    USING (tenant_id = current_setting('app.current_tenant_id')::UUID);
```

### アプリケーションからの呼び出し

アプリケーションの実装イメージについてお話します。三角マークをクリックすると、Spring Bootのモック実装が見れます。Springを使ったことがない方も、なんとなくの流れを感じ取っていただければ幸いです。

::::details 1. リクエストごとに分離された、tenant_idを格納する変数を用意する
    
```java
public class TenantThreadLocalStorage {
    private static ThreadLocal<String> tenant = new ThreadLocal<>();

    public static void setTenantId(String tenantId) {
        tenant.set(tenantId);
    }

    public static String getTenantIdString() {
        return tenant.get();
    }

    public static UUID getTenantId() {
        return UUID.fromString(tenant.get());
    }
}
```

- ThreadLocalクラスを使用すると、スレッドごとに独立した値が保存される変数を格納することができる[^3]
- Spring Bootの場合、リクエストごとにスレッドが立ち上がるため、 1リクエスト = 1スレッドとなる[^4]
- これにより、SpringではThreadLocalのインスタンスをリクエストごとに分離された変数の保管場所として利用できる
::::

::::details 2. リクエストの前処理でリクエストヘッダから tenant_id を取り出し、ストレージにセットする
    
```java
@Component
public class RequestInterceptor implements HandlerInterceptor {
    @Override
    public boolean preHandle(HttpServletRequest request, HttpServletResponse response, Object handler)
            throws Exception {
        // 認証情報を取り出す
        Authentication authentication = SecurityContextHolder.getContext().getAuthentication();

        // 認証情報からtenant_idを取り出す
                Jwt jwt = (Jwt) authentication.getCredentials();
        Map<String, Object> map = jwt.getClaims();
        Object tenantIdObj = map.get("tenant_id");

        // tenantIdがJWTに含まれない場合は401エラーを返す
        if (tenantIdObj == null) {
            response.setStatus(401);
            return false;
        }

        // 1.で用意したTenantThreadLocalStorageにtenantIdをセットする
        TenantThreadLocalStorage.setTenantId(tenantIdObj.toString());

        return true;
    }

    @Override
    public void afterCompletion(HttpServletRequest request, HttpServletResponse response, Object handler, Exception ex)
            throws Exception {
        // Springではスレッドプール内のスレッドは再利用される可能性がある
        // そのためリクエストの処理後、tenantIdをリセットする
        TenantThreadLocalStorage.setTenantId(null);
    }
}
```
::::
    
::::details 3. SQLのコネクションを張る際に呼び出される関数で、1. の変数から tenant_id を取り出し実行時パラメータにセットする
    
```java
public class TenantAwareDataSource extends DataSource {
    // getConnectionをオーバーライドすることでコネクション取得時に任意のコードを実行することができる
    // getConnectionはDBアクセスが走るたびに呼び出される 
    @Override
    public Connection getConnection() throws SQLException {
        Connection connection = super.getConnection();

        // connectionをひらくときに、実行時パラメータに2.でJWTから取り出したtenantIdをセットする
        try (Statement sql = connection.createStatement()) {
            sql.execute("SET app.current_tenant_id = '" + TenantThreadLocalStorage.getTenantIdString() + "'");
        }

        return connection;
    }
}
```

- getConnectionはDBアクセスが走るたびに呼び出される[^5] 
::::
4. 後はRLSのことを意識せず、フレームワークのORMなどでDBアクセスする👌

## RLSの注意点

マルチテナントアーキテクチャを実現する上でたいへん役に立つRLSですが、いくつか注意しなければならない点があります。他にもこれ気をつけたほうがいいよ！というアドバイスお待ちしてます🙏

### 注意点 1. RLSポリシーが適用されないロールがある

デフォルトでは、テーブルの所有者は RLS ポリシーを無視したアクセスが可能です。そのため、テーブルを作成するDBロールとRLSを利用するテーブルロールは分割しましょう。テーブルの所有者に対してもRLSポリシーを適用するには `FORCE ROW LEVEL SECURITY` を利用します。

```sql
// テーブルに対しRLSを有効化する
ALTER TABLE member ENABLE ROW LEVEL SECURITY;

// テーブルの所有者にRLSポリシーを適用する
ALTER TABLE member FORCE ROW LEVEL SECURITY;

\d member;

...
Policies (forced row security enabled):
    POLICY "tenant_isolation_policy"
      TO app_user
      USING ((tenant_id = (current_setting('app.current_tenant_id'::text))::uuid))

```

Nstockの場合、DBマイグレーションを行うDBロールと、アプリケーションでつかうDBロールを分けています。これによって、テーブルの所有者にRLSポリシーが適用されない問題を回避して、アプリケーション用のDBロールにRLSポリシーを適用しています。

:::message
ちなみに、スーパーユーザーや、 `BYPASSRLS` 属性を使用して作成したロールにも、RLSポリシーが適用されません。ただし、スーパーユーザーはセキュリティ上アプリケーションで使わないほうが安全ですし、RLSポリシーを適用するアプリケーション用DBロールに `BYPASSRLS`属性を設定することもないので、あまり気にしなくて良さそうです。
:::

### 注意点 2. RLSポリシーを設定してもインデックスは自動で貼られない

RLSポリシーを設定すると、USING句で指定したカラムを使った問い合わせが頻繁に走ることになります。しかしながらRLSポリシーを作成してもインデックスは自動で貼られません。そのため、RLSポリシーを追加する場合は、インデックスも併せて検討する必要があります。

```sql
\d member;

Table "public.member"
  Column   |          Type          | Collation | Nullable |      Default
-----------+------------------------+-----------+----------+--------------------
 id        | uuid                   |           | not null | uuid_generate_v4()
 name      | character varying(100) |           | not null |
 tenant_id | uuid                   |           | not null |
Indexes:
    "member_pkey" PRIMARY KEY, btree (id)
    // RLSポリシーがある tenant_id にインデックスは貼られていない
Policies:
    POLICY "tenant_isolation_policy"
      TO app_user
      USING ((tenant_id = (current_setting('app.current_tenant_id'::text))::uuid))
```

### 注意点 3. RLSポリシーの設定を誤ると一大事

アクセス権限の管理をRLSに集約することで、変更・確認する箇所を減らすことができるのは、低レイヤーで行レベルアクセス制御が実現できるRLSの利点でしょう。一方でアクセス権限の管理がRLSに集約される分、RLSの設定を誤ると一大事です。

Nstockではプルリクエストのレビュー項目に「RLSポリシーが適切に設定されているか」をいれたり、月に一度のエンジニア定例作業のなかで、RLS設定の棚卸ししたりしています。RLSの設定内容を確認する自動テストも今後追加していく予定です。

## さいごに

**RLSはいいぞ**

次回はB2B2E SaaSにおけるRLSを使ったDB設計についてお話しようと思います。これからもRLSの試行錯誤について発信していくので、よかったらまた見に来てください **👋**

最後に、Nstockではエンジニアを募集しています！スタートアップを皮切りに日本を変えていきませんか？

[カジュアル面談](https://open.talentio.com/r/1/c/nstock/pages/73091)から気軽にお話しましょう🤞

https://speakerdeck.com/nstock/we-are-hiring

https://open.talentio.com/r/1/c/nstock/pages/70402

[^1]: https://www.postgresql.org/docs/current/ddl-rowsecurity.html

[^2]: https://www.postgresql.org/docs/current/sql-syntax-lexical.html#SQL-SYNTAX-IDENTIFIERS

[^3]: https://docs.oracle.com/javase/jp/17/docs/api/java.base/java/lang/ThreadLocal.html

[^4]: Spring BootではServletコンテナとして、Tomcat、Jeety、Undertowをサポートしています(デフォルトはTomcat)[^6] 。Servletコンテナでは各HTTPリクエストはリクエストごとにスレッドが立ち上がります[^7]。ただし、リアクティブプログラミングモデル([Spring WebFlux](https://docs.spring.io/spring-framework/reference/web/webflux.html))を利用する場合、Servletコンテナの代わりに[Reactor](https://github.com/reactor/reactor)が使われます。この場合、1スレッドで複数のリクエストが処理される可能性があります。このように、Spring Bootを利用していても1リクエスト=1スレッドとはならないケースがあります。

[^5]: Spring Frameworkでは、データベース接続は通常、トランザクション単位で管理されます。つまり、トランザクションが開始されるときにデータベースコネクションが取得され、トランザクションが終了（コミットまたはロールバック）されるときにコネクションがリリースされます[^8]。

[^6]: https://docs.spring.io/spring-boot/docs/current/reference/htmlsingle/#web

[^7]: https://download.oracle.com/otndocs/jcp/servlet-4-final-spec/index.html

[^8]: https://docs.spring.io/spring-framework/docs/4.2.x/spring-framework-reference/html/transaction.html#transaction-local