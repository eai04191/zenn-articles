---
title: "gh cliでGitHub Organizationのチームメンバーを別のチームにコピーする"
emoji: "🚛"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["githubcli"]
publication_name: "ncdc"
published: true
---

https://docs.github.com/ja/organizations/organizing-members-into-teams/about-teams

PRのレビューにGitHubのチーム機能を使ってみようと思い、やりようを考えたところ似たようなチームを複数作ることになりました。

GitHubのWeb上ではメンバーのコピーなどはできず、一人一人追加するのが果てしなく面倒だったので、gh cliを使ってチームメンバーを別のチームにコピーする方法を考えました。

以下のようなコマンドでできます

```bash
TEAM_FROM="myorg/teams/team-from"
TEAM_TO="myorg/teams/team-to"

gh api orgs/$TEAM_FROM/members | \
jq -r '.[].login' | \
xargs -I {} echo gh api --method PUT orgs/$TEAM_TO/memberships/{}
```

TEAM_FROMとTEAM_TOは先にチームを作ってURLから取得します。

このコマンドは事故を防ぐためにechoを挟んでdry-runにしています。実行するとTEAM_FROMのメンバー分、gh apiコマンドが出力されます。

```bash
gh api --method PUT orgs/myorg/teams/team-to/memberships/foo1
gh api --method PUT orgs/myorg/teams/team-to/memberships/foo2
gh api --method PUT orgs/myorg/teams/team-to/memberships/foo3
gh api --method PUT orgs/myorg/teams/team-to/memberships/foo4
gh api --method PUT orgs/myorg/teams/team-to/memberships/foo5
gh api --method PUT orgs/myorg/teams/team-to/memberships/foo6
...
```

といった具合で実行予定のコマンドが出力されるので、問題なければechoを外して実行すれば実際にgh経由でAPIが呼ばれてメンバーが追加されます。

## 参考

https://docs.github.com/ja/rest/teams/members?apiVersion=2022-11-28#add-or-update-team-membership-for-a-user
