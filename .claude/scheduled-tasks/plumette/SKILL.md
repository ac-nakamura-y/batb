---
name: plumette
description: 中村宛の依頼の Issue 化
schedule: "30 10-19 * * 1-5"
timezone: Asia/Tokyo
---

# Todo Issue Sync

## Overview

議事録と Slack に届いた中村勇士宛の依頼を集め、中村に確認したうえで Linear の Issue にする。依頼はそれぞれの場所に残ったままで、ここで作るのは着手の入口となる Issue である。議事録の所在は `~/batb` の資料データベースが持ち、`~/batb/bin/batb` がその操作コマンドである。本文は Google Doc にだけあるので、読むときは Google Drive から取る。

Backlog は対象にしない。依頼は Backlog の課題としてそこに残り、担当も期限もその課題が持つ。Linear に写すと同じ作業が 2 か所に並ぶ。受け皿を持たない議事録と Slack だけが対象である。

処理は起票済みの確認、候補の収集、未起票の抽出、候補の提示、返信を受けての起票の順に進む。

```mermaid
flowchart LR
  listIssues[起票済み] --> collect[収集]
  collect --> findNew[未起票]
  findNew --> propose[提示]
  propose --> reply[返信]
  reply --> saveIssue[起票]
```

## Registered issues

最初に、起票済みの依頼と、起票しないと返信された依頼を調べる。Linear の `list_issues` を共通の引数で `3` 回呼び、結果を合わせて起票済みの一覧とする。

```yaml
team: marutto-ops
assignee: me
limit: 250
fields: ["title", "description", "url", "statusType"]
```

| argument | 取る Issue |
| :-- | :-- |
| `createdAt: -P14D` | 直近 `14` 日に作られたもの。完了やキャンセルの済んだものも含む |
| `state: started` | 着手中のもの。作成日は問わない |
| `state: unstarted` | 未着手のもの。作成日は問わない |

進行中の作業は `14` 日より前に作られていることもあるので、着手中と未着手は作成日を問わずに取る。`list_issues` の `query` は曖昧検索で、本文に書かれた URL やトークンを拾わないため使わない。

起票しないと返信された依頼は、見送りの記録 `~/batb/tmp/plumette-declined.txt` から読む。記録は 1 行に 1 件で、出典の文字列と案の title をタブで区切って書いてある。ファイルがなければ、見送った依頼はまだない。

候補は次の 2 つの照合にかけ、どちらかに当たれば示さない。表の「未完了」は、`statusType` が `completed` でも `canceled` でもないことを指す。

| check | 照合に使う文字列 | 照合先 |
| :-- | :-- | :-- |
| 出典 | 議事録は Google Doc の ID、Slack は permalink の `p` で始まるタイムスタンプ | 一覧のすべての Issue と見送りの記録 |
| ファイル | 依頼の文面が指す Google のスライド・スプレッドシート・ドキュメントの ID | 一覧の未完了の Issue |

出典の照合は、同じ依頼を二度起票せず、見送った依頼を再び示さないためにある。1 つの議事録や投稿から別々の依頼が出るため、出典の文字列が一致した Issue や記録のうち、同じ依頼を扱うものがあるときだけ当たりとする。返信で触れられなかった案は照合に当たらず、次の実行でも示される。ファイルの照合は、進行中の作業に含まれる依頼を別の Issue にしないためにあり、出典のリンクを持たない手作りの Issue もこれで拾える。議事録の Google Doc は出典なので、ファイルの照合には使わない。1 つの会議から別々の宿題が出るためである。

一覧の本文は長いと途中で切れる。出典のリンクは本文の先頭にあるので、出典の照合は一覧の本文で足りる。ファイルの照合にかける候補があるときは、本文が切れた未完了の Issue を先に `get_issue` で全文にする。

文字列が一致しなくても、同じ作業の Issue が既にあるなら起票しない。`PR #N のレビュー依頼` は別の仕組みが作るため、PR のレビューだけを求める依頼がこれにあたる。

## Scope

対象は、2 つの入口それぞれで過去 `2` 日に届いたもののうち、まだ起票されていない依頼である。この実行は毎時走るため、`2` 日あれば数回の失敗を挟んでも取りこぼしは起きない。遅れを理由に範囲を広げたり、候補から外したりしない。

議事録は資料データベースから取る。`created` が過去 `2` 日で、出典が Google Doc のものが対象である。

```bash
~/batb/bin/batb query --limit 50
```

議事録の本文は、出典の URL から Google Drive の `read_file_content` で読む。

Slack は自分宛のメンションを検索する。自分の Slack user id は `slack_search_public_and_private` の説明に書かれた値を使い、他の場所から持ち込まない。`limit` の上限は `20` なので、結果が過去 `2` 日より古くなるまで `cursor` を辿る。

```yaml
keywords: ["<自分の user id のメンション>"]
filters: "after:<2 日前の日付>"
natural_language_query: ""
response_format: detailed
include_context: false
```

## Selection

候補のうち、中村がこれからやることだけを起票する。入口ごとの見分け方を次に示す。

| source | 起票する | 起票しない |
| :-- | :-- | :-- |
| 議事録 | 中村が担当と書かれた宿題・持ち帰り | 担当が書かれていない項目、決定事項の記録 |
| Slack | 中村への依頼・確認・レビュー要請 | cc だけの投稿、お礼や相槌、bot の定型投稿 |

Slack は投稿だけでは依頼かどうか分からないことがある。スレッドの文脈が要るときは `slack_read_thread` で親から読む。

## Proposal

この実行では質問のツールを使えない。候補を Issue の案として表にまとめて示し、実行を終える。中村はこのセッションに返信し、起票する案と起票しない案を番号で伝える。

```markdown
| # | title | project | 出典 | 補足 |
| --: | :-- | :-- | :-- | :-- |
| 1 | 〇〇の資料を作る | [FDE] 制作プロセス改善 | [9/30 FDEデイリー](https://docs.google.com/document/d/<doc_id>/edit)：〇〇の資料作成の宿題 | 担当の書き方が曖昧 |
```

title と project は Creation に従って用意し、合う project がなければ「なし」と書く。出典には、リンク付きの出典名と依頼の要約を書く。補足には、宛先や担当が読み取れないなど判断に要ることを書き、推測で埋めない。

表のあとに、ファイルの照合で飛ばした候補を、出典の URL と同じファイルが書かれた Issue の URL とともに挙げる。候補が `1` 件もなければ、その旨を `1` 行で述べる。

## Reply

中村の返信を受けて、案ごとに次のとおり扱う。

| reply | action |
| :-- | :-- |
| 起票すると選んだ | Creation に従って Issue を作る |
| 起票しないと伝えた | 見送りの記録に追記する |
| 触れていない | 何もしない。次の実行で再び示す |

```bash
printf '%s\t%s\n' '<出典の文字列>' '<title>' >> ~/batb/tmp/plumette-declined.txt
```

作った Issue の title と URL を、案ごとに示す。

## Creation

Issue は Linear の `save_issue` で作る。同じ案が別の実行のセッションにも出ているため、作る前に Registered issues の出典の照合をやり直し、同じ依頼の Issue が既にあれば作らない。返信に修正の指示があれば、それに合わせて直してから作る。既定値を次に示す。

| field | value |
| :-- | :-- |
| team | `marutto-ops` |
| project | 内容に合う進行中のプロジェクト |
| assignee | `me` |
| state | `Triage` |
| priority | `3` |

project は `list_projects({ team: "marutto-ops", state: "started" })` の一覧と、出典から読み取れるクライアント名やプロジェクト名を突き合わせて選ぶ。迷ったときは `~/batb/bin/batb term infer "<タイトルや文面>"` で用語を確認する。合うものがなければ project を付けない。

title は何をするかを一文で書く。description は出典の引用から始め、リンクを引用の中に埋める。次の実行の重複確認がこの引用を手がかりにするため、リンクは必ず残す。

```markdown
> **戸田弥希** [2026-09-25 09:23](<https://activecorehq.slack.com/archives/C0B7E2BARH7/p1790295838570279>)
>
> 以下2点お願いいたします
> ・来週月曜の内部定例ですがキャンセルでお願いします
```

引用のリンク先は、議事録なら `https://docs.google.com/document/d/<doc_id>/edit`、Slack なら検索結果の permalink とする。引用だけで作業内容が伝わらないときは、引用の下に背景と完了条件を数行で足す。

Issue の作成と見送りの記録への追記以外はしない。Slack へ返信せず、Git で管理するファイルの編集やコミットも行わない。
