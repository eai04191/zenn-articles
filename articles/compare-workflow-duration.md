---
title: "ubuntu-slimに切り替える際に実行時間がどれほど伸びるか比較する"
emoji: "💰"
type: "tech"
topics: [githubcli, githubactions]
publication_name: "ncdc"
published: true
---

先日GitHub Actionsに新しいrunnerとしてubuntu-slimが追加されました。

https://zenn.dev/my_vision/articles/actions-ubuntu-slim

スペックが下がるとはいえコストが1/4まで減るのはとても魅力的ですが、jobの実行時間がだらだらと伸びてしまっては困ります。実際に動かして許容できる範囲か確認してみました。

観点は2つです。

1. jobが実行時間以外今まで通り動くか
1. それぞれのjobの実行時間がどれくらい伸びるか

## 比較手順

今回はPR契機のjobを比較します。まずはPRを2つ作ります。

- `blank`: defaultブランチから切って`--allow-empty`でコミットを作っただけのブランチ
- `ci/ubuntu-slim`: すべてのrunnerをubuntu-slimに変更したブランチ

ubuntu-slimはプリインストールのツールが減っているうえに一覧が公開されていないと聞いていたのでそのままで動くか心配だったのですが、私の作ったものはすべてあっさりと動きました。少なくともnodeやcorepackは入っているようです。1つ目の観点はクリアしました。

## 実行時間を比べる

web uiでにらめっこするのは大変なので、gh cliでなんとかできないかなと思ったらなんとかできそうでした。

まず`gh pr checks`でPRのworkflow実行結果を取得できます。`--json`オプションで必要な情報を絞り込めるので、workflow名、開始時間、終了時間を取得し、jqで実行時間を計算、awkで比較します。えらそうに言っていますが全部geminiに書いてもらいました。

macOS 15のzshで動作を確認しました。たぶんbashでも動くと思います。動かなかったらgeminiとかに聞いてください。

```zsh
(
  BRANCH_BEFORE="blank"
  BRANCH_AFTER="ci/ubuntu-slim"

  get_check_durations() {
    local branch_name="$1"

    gh pr checks "$branch_name" --json name,startedAt,completedAt | \
    jq -r '
      .[] |
      ( .completedAt | fromdateiso8601 ) as $end |
      ( .startedAt | fromdateiso8601 ) as $start |
      ($end - $start) as $duration |
      [ .name, $duration ] | @tsv
    ' | \
    sort -k1,1
  }

  ( \
    echo -e "Job Name\tBefore(sec)\tAfter(sec)"; \

    join -t $'\t' -a 1 -a 2 -e "N/A" -o 0,1.2,2.2 \
      <( get_check_durations "$BRANCH_BEFORE" ) \
      <( get_check_durations "$BRANCH_AFTER"  ) \
  ) | \
  awk '
    BEGIN { OFS="\t"; FS="\t" }
    NR == 1 { print $0, "Diff(%)"; next }
    {
      before_str = $2; after_str = $3; diff_percent = "N/A"
      if (before_str != "N/A" && after_str != "N/A") {
        before = before_str + 0; after = after_str + 0
        if (before == 0) {
          diff_percent = (after == 0) ? "0.00%" : "+Inf%"
        } else {
          percent = ((after - before) / before) * 100
          diff_percent = sprintf("%+.2f%%", percent)
        }
      }
      print $0, diff_percent
    }
  ' | \
  column -t -s $'\t'
)
```

実行結果は以下のようになりました。

```
Job Name                              Before(sec)  After(sec)  Diff(%)
Analyze (javascript-typescript)       72           71          -1.39%
build / build                         39           85          +117.95%
calc_build_size / calc_build_size     58           112         +93.10%
check / check                         49           114         +132.65%
check_production / post               3            4           +33.33%
check_production / pre                3            5           +66.67%
check_production / run-check / check  55           135         +145.45%
deploy_preview / deploy               62           100         +61.29%
pinact / pinact                       16           13          -18.75%
reviewdog / eslint                    55           106         +92.73%
test / test                           23           62          +169.57%
```

Analyze (javascript-typescript), pinact / pinactはcodeqlとorgのrulesetで追加されたものなので今回の比較対象ではありません。

その他のjobは実行時間が伸びています。vCPUが半分になったので2倍になるのは予想通りです。伸び率はjobによってまちまちですが、だいたい1.5倍から2.5倍の間に収まっています。

時間単価が1/4なので実行時間が4倍を超えるとコストが下がらない、なんなら時間分損していることになりますがそこまではいかないので、基本的にubuntu-slimを使うでコストは下がるといえそうです。元が数秒のjobはほとんど影響がないので移行しても問題ないでしょう。

testやbuildのようなCPUパワーがモロに効いて、かつ依存関係の上流にあるjobは悩ましいところです。どうせPR出したらコーヒーでも淹れに行くので1分でも2分でも変わらないだろという説もあります。またmainのようなマージした後にある程度(5分くらい)時間がかかってもいいけど動いて欲しいところにはubuntu-slimを使うのも悪くないのかもしれません。

## まとめ

- `gh pr run`の結果を比較できるコマンドを作った
    - 逆にハイスペックランナーとの比較にも使えるかも
- ubuntu-slimはわりと動く
- ubuntu-slimはコスト1/4, 時間だいたい2倍

## 宣伝

NCDCではCI/CDの実行時間とコストを少しでも減らしたいエンジニアを募集しています。
