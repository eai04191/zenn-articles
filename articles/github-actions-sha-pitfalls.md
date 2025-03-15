---
title: "サードパーティアクションはSHAで指定すれば安心？残念ながら、いいえ"
emoji: "🔒"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [githubactions]
publication_name: "ncdc"
published: false
---

## TL;DR

- SHAで指定されたアクション自体がタグで別のアクションを呼び出していたら、その呼出先は呼び出した時点のタグの向き先になるから
    - タグ指定をしていないcompositeアクション、（ランタイムでevalなどをしないのは前提として）nodeやdockerタイプのアクションならSHA指定でおそらく安全

---

2025/3/15くらいに、GitHub Actionの有名なサードパーティアクションが侵害される事件がありました

- https://semgrep.dev/blog/2025/popular-github-action-tj-actionschanged-files-is-compromised/
- https://www.stepsecurity.io/blog/harden-runner-detection-tj-actions-changed-files-action-is-compromised
- https://news.ycombinator.com/item?id=43367987

`tj-actions/changed-files`が侵害されて、機密情報をログに出力する悪意のあるコードが混入してしまったというものです。

実行手順としては`tj-actions/changed-files`のすべてのタグ指定が侵害されたコミット(`0e58ed8671d6b60d0890c21b07f8835ace038e67`)を向くように書き換えられたため、`uses: tj-actions/changed-files@v45`のような指定をしているworkflowは、侵害されたコードを実行してしまうことになっていました。現時点ではリポジトリごと（ユーザーごと？）削除されているので実行されません。

## actionのタグは、gitのタグ。gitのタグは書き換えられる

GitHub Actionsでサードパーティアクションを呼び出すときに何が行われているのか、おさらいしておきましょう。

私たちが普段なにげなく使っているGitHub Actionsのworkflowで`uses: actions/checkout@v4`のように書くと、これは`actions/checkout`リポジトリのタグ`v4`を指定していることになります。

（正確にはブランチも指定できますが、一般的にはタグを指定することが多いので、ここではタグに絞って説明します）

これは一見セマンティックバージョニングに基づいているように見えますが、実はそうではありません。GitHubのドキュメントでは[セマンティックバージョニングを使用することを推奨しています](https://docs.github.com/en/actions/sharing-automations/creating-actions/about-custom-actions#using-tags-for-release-management)が、gitのタグとして許されるものなら何でも使えます。しかしセマンティックバージョニングのように使えたほうが便利であることをみな知っているので、サードパーティアクションの作者が気を利かせてやってくれているだけなのです。

具体的な例を考えてみましょう。

私たちのworkflowに以下のような記述があるとします。

```yaml
- name: Checkout
  uses: actions/checkout@v4
```

これを実行したとき、runnerは[actions/checkout](https://github.com/actions/checkout)リポジトリのv4タグを指すコードをダウンロードしてきます。今v4は……[v4.2.2である`11bd719`](https://github.com/actions/checkout/commit/11bd71901bbe5b1630ceea73d27597364c9af683)を指しているようです。

さて、時間が経ち、actions/checkoutをより良くするために、書き込み権を持っている人が変更を加え、新しいバージョン v4.2.3 (`423aaaa`) をリリースしたとします。さらに、このとき**v4タグがv4.2.3のコミット`423aaaa`を指すように書き換えてくれました**。

再度私たちのworkflowを実行すると、今度はactions/checkoutのv4.2.3がダウンロードされます。今v4が指しているのは`423aaaa`だからです。
私たちは既存のworkflowに何も変更を加えることなく、新しいバージョンを使うことができました。みんながハッピーです。めでたしめでたし。

このように、タグを書き換えられるということは適切に使えば大変便利なことなのです。

## （つかの間の）安心を手に入れよう

ええ、あなたのおっしゃりたいことはわかります。タグが書き換えられることによって、私たちのworkflowがいつの間にか悪意のあるコードを実行してしまったらどうしよう、ということですね。

ご安心ください。GitHubは私たちの気持ちをわかっています。タグではなくSHAで指定することもできるのです。

> We strongly recommend that you include the version of the action you are using by specifying a Git ref, SHA, or Docker tag.
>
> https://docs.github.com/en/actions/writing-workflows/workflow-syntax-for-github-actions#jobsjob_idstepsuses

つまり、さきほどのworkflowは以下のように書き換えることができます:

```yaml
- name: Checkout
  uses: actions/checkout@11bd71901bbe5b1630ceea73d27597364c9af683 # v4.2.2
```

これで、actions/checkoutのv4.2.2を指すコードがダウンロードされることが保証されます。v4の参照先がどうなろうと、v4.2.2を指すコードがダウンロードされることになります。

これで安心ですね！

なおこのように書き換えるのがめんどくさい場合はRenovateや[pinact](https://github.com/suzuki-shunsuke/pinact)でいい感じにできるそうです。

## 安心ではないことを確かめよう

本題です。

ここまではいろいろな所に書いてあることです。しかし、この方法でも安心ではないことがあります。

それは、参照先のアクションが他のアクションをタグで呼び出している場合です。

私たちのworkflowがいくらSHAで指定していても、そのアクションが他のアクションをタグで呼び出している場合、その呼び出されたアクションは呼び出された時点のタグの向き先になってしまいます。

一応試してみましょう。リポジトリを3つ用意しました:

- https://github.com/eai04191/test-good-action
    - 誰かが作ったサードパーティアクションです
    - このリポジトリが侵害されるという体で進めます
- https://github.com/eai04191/test-superturbo-action
    - 誰かが作ったサードパーティアクションです
    - 入れるだけでworkflowが超速くなると謳うアクションです
    - 内部で`eai04191/test-good-action@v1`を使用しています
- https://github.com/eai04191/test-action-user
    - workflowを持っています
    - わたしたちはこのリポジトリのみにアクセス権を持っているとします
    - `eai04191/test-superturbo-action`をSHAで指定して使用します

侵害される前の各ファイルは以下のようになっています:

```yaml:test-good-action\action.yml
name: "Good Action"
description: "This is a good action. (This is a Demonstration for the GitHub Actions Dependency Problem)"
runs:
  using: "composite"
  steps:
    - run: echo "Hello, World!"
      shell: bash
```

```yaml:test-superturbo-action\action.yml
name: "SuperTurbo"
description: "This action accelerates the workflow. (This is a Demonstration for the GitHub Actions Dependency Problem)"
runs:
  using: "composite"
  steps:
    - uses: eai04191/test-good-action@v1

    - run: echo "Speed up!"
      shell: bash
```

```yaml:test-action-user.github\workflows\run.yml
name: Run Workflow

on: workflow_dispatch

jobs:
  run:
    runs-on: ubuntu-latest
    steps:
      - name: Use Superturbo
        uses: eai04191/test-superturbo-action@5fc4223282d36d4fe373fd8bf34df99f1dcecd67

      - run: echo "Done"
        shell: bash
```

eai04191/test-good-actionは`v1`タグで`900d111badad0910c520ef3f4a5927b8f9217b61`を指しています。

![](/images/github-actions-sha-pitfalls/900d111.png)

実行結果は以下のようになりました。いいですね。

https://github.com/eai04191/test-action-user/actions/runs/13875298166/job/38826836626

```log:Set up Job 抜粋
Prepare all required actions
Getting action download info
Download action repository 'eai04191/test-superturbo-action@5fc4223282d36d4fe373fd8bf34df99f1dcecd67' (SHA:5fc4223282d36d4fe373fd8bf34df99f1dcecd67)
Getting action download info
Download action repository 'eai04191/test-good-action@v1' (SHA:900d111badad0910c520ef3f4a5927b8f9217b61)
Complete job name: run
```

Set up Jobで`5fc4223282d36d4fe373fd8bf34df99f1dcecd67`と`900d111badad0910c520ef3f4a5927b8f9217b61`がダウンロードされていることを覚えておいてください。

```log:Use Superturbo 抜粋
▶ Run eai04191/test-superturbo-action@5fc4223282d36d4fe373fd8bf34df99f1dcecd67
▶ Run eai04191/test-good-action@v1
▶ Run echo "Hello, World!"
Hello, World!
▶ Run echo "Speed up!"
Speed up!
```

---

さて、侵害が発生しました。`eai04191/test-good-action`リポジトリが侵害され、新しいコミットがpushされ、`v1`タグは`badbad0fae80a5c446118e22f35d25aef6d33fe1`を指すように書き換えられました。

```diff:test-good-action\action.yml
 runs:
   using: "composite"
   steps:
-    - run: echo "Hello, World!"
+    - run: echo "Hello, Evil World!"
       shell: bash
```

![](/images/github-actions-sha-pitfalls/badbad0.png)

そして私たちのworkflowを再度実行してみましょう。がんばってSHAで指定したので何も変わってないといいのですが！

https://github.com/eai04191/test-action-user/actions/runs/13876060873/job/38828506686

```log:Set up Job 抜粋
Prepare all required actions
Getting action download info
Download action repository 'eai04191/test-superturbo-action@5fc4223282d36d4fe373fd8bf34df99f1dcecd67' (SHA:5fc4223282d36d4fe373fd8bf34df99f1dcecd67)
Getting action download info
Download action repository 'eai04191/test-good-action@v1' (SHA:badbad0fae80a5c446118e22f35d25aef6d33fe1)
Complete job name: run
```

```log:Use Superturbo 抜粋
▶ Run eai04191/test-superturbo-action@5fc4223282d36d4fe373fd8bf34df99f1dcecd67
▶ Run eai04191/test-good-action@v1
▶ Run echo "Hello, Evil World!"
Hello, Evil World!
▶ Run echo "Speed up!"
Speed up!
```

あーあ、`eai04191/test-good-action@v1`が`badbad0fae80a5c446118e22f35d25aef6d33fe1`を指すように書き換えられてしまったので、悪意のあるコードが実行されてしまいました。

## どうすればよかったのか

残念ながら、この問題に対する手っ取り早い特効薬はありません。

私の思いつく範囲では:

- **可能な限り、サードパーティアクションを使わない**
    - shellでできないか？
    - Node.jsやPythonのスクリプトファイルをorg内に置いて呼び出すだけでは足りないか？
- どうしても使う場合は
    - 心中する覚悟がある公開元のみ使う
        - actions/ など
    - 最低限、何をしているのかは読んでおく
        - 他のアクションをタグで呼び出していないか
        - evalやcurlしていないか
    - 呼び出しがなければSHAで指定する
    - 呼び出しをしていて、どうしても使いたい場合はforkしてSHAで呼び出したり、処理を変更する

といった方法が考えられます。いずれも利便性とのトレードオフがありますが、どうしても使いたい場合は慎重に検討してください。

## 広告

NCDCでは、安全なサプライチェーンをチームで模索しながらプロダクトを開発したいエンジニアを募集しています。
