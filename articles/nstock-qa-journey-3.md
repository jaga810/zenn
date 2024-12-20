---
title: "Nstock QAの旅 #3 インシデントレベルを考えるワークショップ"
emoji: "😽"
type: "idea" # tech: 技術記事 / idea: アイデア
topics: ["QA"]
published: true
publication_name: "nstock"
---

こんにちは！jagaです 🥔

私が所属するNstockでは現在、QAエンジニアがいません。
QAエンジニアの採用活動を進めていますが苦戦しています。

そこで、令和トラベルでQAエンジニアをされている[miisan](https://x.com/mii________san)の協力を得て、できることから始めています。

## これまでに実施した取り組み
これまでNstockで実施してきた取り組みです。

1. [品質に対する思いを相互理解するワークショップ](https://zenn.dev/nstock/articles/nstock-qa-journey-1)
2. [品質レベルを考えるワークショップ](https://zenn.dev/nstock/articles/nstock-qa-journey-2)
3. インシデントレベルを考えるワークショップ
4. QAエンジニアの求人の更新

今回は「インシデントレベルを考えるワークショップ」についてお話しします。

## 背景

前回実施した[品質レベルを考えるワークショップ](https://zenn.dev/nstock/articles/nstock-qa-journey-2)では、品質を考えるうえでの軸になりそうなキーワードが洗い出せました。

- セキュリティインシデント
- 代替手段
- 法令遵守
- Nstockとの契約内容とのアライン
- 体験の希少性
- 影響範囲
- タイミング

今回は普段の意思決定に品質レベルの考え方を活かせるよう、「インシデントレベル」を考えることにしました。

## 目的

「より実践的な品質レベルを作り、品質を管理できる状態にする」ために「Nstockにおけるインシデントレベル」を言語化すること

## やり方

**参加者**

株式報酬SaaS Nstockの開発・運用に関わるメンバー全員(PdM、デザイナー、CS、エンジニア)

**進め方**

1. インシデントレベルの概要説明 by miisan
2. 過去のインシデントを全て書き出す
3. 過去のインシデントをインシデントレベルの叩き台に当てはめてみる
4. インシデントレベル自体を洗練する

適宜、miisanにコメントをもらったり、互いに質問したりしながら進めました。


## 当日の様子

### インシデントレベルの概要説明
最初に、なぜインシデントレベルを考えると良いか、miisanから解説していただき、目線合わせをしました。

> エンジニアが主体的に色々考える組織は、品質が高いプロダクト出せるのは必然。前回のワークショップで挙げられた観点は、品質特性の網羅率が非常に高かった。

> では良い状態の今、なぜ品質について言語化すべきなのか？ → 品質の重要性は理解していても、バックグラウンドや経験値によって濃淡が変わる可能性があるから。今後スケールするにあたって、一つの指針を持つことで良い文化は当たり前となっていく。
![品質保証を考える上でのポイント「品質保証をする」 が何かは、会社やプロダクト、 サービスの特性によって変わるエンタメサービスと金融サービスで同じ品質基準になることが適切なのか?「私たちは良いプロダクトを作っています!」 の“良いプロダクト”・”良い品質”について、チームで同じ解釈で語れるか?異なる解釈を持って進むと、 バラバラのゴールに辿り着いてしまう「カスタマーにとっての価値を作る」 とは一体何か?価値の解釈が異なると、 優先順位に齟齬がでる](/images/nstock-qa-journey-3/qa-tips.png)

> 「テストした？」って何か？
![無限の時間を使わずに、 「テストした」と言える状態とは? 1.重大なインシデントを引き起こす欠陥がないこと 2. 現在の状態で十分な価値を実現できていること 3. プロダクトの価値が残存リスクを上回ること 4. リリースする利益が遅らせる損害を上回ること](/images/nstock-qa-journey-3/enough-testing.png)

> 言い換えると、”重大なインシデント”の基準がなければ十分なテストをしたとは言えないし、”十分な価値”がわかっていなければ本当にリリースして良いのかはわからない🙅‍♀️ なぜQAエンジニアは、リリースOKの最終ジャッジメントを出せるかというと、明確な軸があるから。

> なので、「重大なインシデントとは何か？」や「目指したい方向性の基準値」を揃える必要があるのです👌

QAのプロにこれからやることの必要性をわかりやすくお話しいただくことで、全員の向かう方向が揃えられました。

### 過去のインシデントを全て書き出す

プロダクト開発初期から社内インシデント事例を貯める、Postmortem DBをつくってたので、簡単に洗い出すことができました。

![Postmortem DB](/images/nstock-qa-journey-3/postmortem.png)
*Postmortem DB*

### 過去のインシデントをインシデントレベルの叩き台に当てはめてみる

![figjamのボード](/images/nstock-qa-journey-3/ticker.png)
*付箋の色ごとの意味*

![figjamのボード](/images/nstock-qa-journey-3/figjam.png)
*共有に使ったFigJam*

インシデントレベルをマッピングする中で軸としてできたものは以下です。

- P1: 即時修正
    - 影響が全テナントにあり、コミュニケーションコストが高い
    - 権利者に対して案内が必要
    - 代替手段がない
    - 代替手段があるが、お客様側 or CSで実行できない
- P2: 営業日+1日中に修正
    - 描画の崩れ
    - 代替手段がありお客様 or CSが実行できる
    - 代替手段はないが、対応までの期限が遠い
- P3: 次サイクルまでの対応
    - (優先度が低く、Postmortemに残っていないこともあり、基準の議論が難しかったです。バックログのBugラベルを持つチケットをピックアップしたらよかったです...)

前回出てきた観点に比べると、実例を出すことでだいぶ絞られました。

### インシデントレベル自体を洗練する
過去のインシデントをインシデントレベルにマッピングするのと並行して、インシデントレベルの枠組み自体についても議論しました。

**叩き台時点でのインシデントレベル**
| インシデントレベル | 対応 |
| --- | --- |
| P0 | 全社で対応 |
| P1 | 開発をストップし、即時対応 |
| P2 | 次のサイクルで対応 |
| P3 | 対応しない |

**ディスカッションを経たインシデントレベル**

| レベル | 対応 |
| --- | --- |
| P1 | war room設置。即時修正。 |
| P2 | 営業日+1日中に修正。 |
| P3 | 次サイクルまでに対応。 |

- P0とP1は対応が同じ（開発ストップ&即時対応）なので、統合しました。
- 対応しないもの(元のP3)はインシデントではないので、インシデントレベルから除きました。
- 即時対応と次のサイクル(最長一週間)までに対応の間に、 `営業日+1日中に修正` が増えました。業務時間外に集まって解決するほどじゃないけど、早めに治したい、みたいな温度感です。


### インシデントレベルの決定フロー
当日話した内容をもとに、筆者の方でインシデントレベルの決定フローを作りました。

![figjamのボード](/images/nstock-qa-journey-3/decision-tree.png)
*インシデントレベルの決定フロー*

コミュニケーションコストが高いか、重要業務とは何か？がわかりやすいよう、以下のような補足をつけています。

- コミュニケーションコストが高い例
    - 影響が全テナントにある
    - 権利者に対して案内が必要
- 重要業務の例
    - 会社と従業員でSOを付与する契約を締結する
    - 税務署への提出が必要な書類を生成する
    - SOを行使して株式を取得する

また、一番大事な約束事として、意思決定フローにない理由で、P1じゃない？となったらP1でOK！ということにしています。

これからの障害対応の経験や、プロダクトの成長に伴って、この決定フローは変えていく予定です。

## 振り返り

### よかったこと
- 実際のインシデントについて話すことで、インシデントレベルを考える上での観点がほどよく絞られた
- 品質を考える軸となり、実務でも使える尺度「インシデントレベル」の共通認識が得られた

### miisanコメント
- 普段話さない価値観がでる
- これからどういう人と働くか考えるきっかけにもなる
- チームで目標設定ができると、ネクストアクションとしていいかもしれない

### ワークショップを経て変わったこと

**Postmortem DBのカラム構成**
品質の分析をする際に、よりよいデータが使えるよう、いくつかのカラムを追加・修正しました。

- 障害検知時に設定したインシデントレベルと、障害対応振り返り時に精査したインシデントレベルを追加
- 不具合要因(仕様考慮漏れ、仕様実装漏れ、実装考慮漏れ、運用不備、デグレ)を追加
- 発生期間のカラムを、発生、検知、復旧と別の日時カラムに分割

**インシデント対応テンプレート**

インシデント対応テンプレートに、インシデントレベルに応じた動き方を追記しました。

![インシデントレベルに応じた動き方](/images/nstock-qa-journey-3/response.png)
*Postmortem DBのテンプレートの一部抜粋*

これによってインシデント時に何をすべきか？が明確になり、動きやすくなりました。


### 次は...
品質を定量的にマネジメントができる状態になったので、品質管理や分析などに活用していきたいと考えています。

考えられると良さそうなこととして、例えば以下が挙げられます。

- チームでインシデントの発生頻度をどのくらいに抑えるか
- いかに復旧スピードを高速化し、被害を最小限に防げるか
- インシデント以外の側面で、どのような品質のメトリクスがあるとよいか


### 一方で

これまでの数回のワークショップによって、品質やインシデントレベルの言語化に取り組んできました。

エンジニアの価値観や、組織として目指したい品質が一定見えてきたことで、Nstockの求めるQAエンジニア像がより明確になりました。

それによって、Nstockの求めるQAエンジニア像と、QAエンジニアの募集要項の乖離に気付いたのです。

これでは本当に来て欲しいQAエンジニアに応募していただけません。

そこで、QAの募集要項をアップデートすることにしました。

Nstock QAの旅は続く...

## We are hiring!

Nstockでは、QAエンジニアを募集しています！

Nstockの開発メンバーは品質の改善をもっとやっていきたいと思っていて、学ぶ姿勢があり、とりあえずはできることからやっていっています。

が、まだまだ道半ばです。

各開発チームが品質保証に関する知識と技術を取得し、自立して活用できるよう、一緒に伴走していただけませんでしょうか？

ちょっとでも興味が湧いた方は、気軽にカジュアル面談にお申し込みください!

https://herp.careers/v1/nstock/THyyh62L_Zkc

https://herp.careers/v1/nstock/ITMAoU-QgtQt