# BatB

## Overview

BatB は会議・チャット・ドキュメントの文脈を Agent に渡すためのワークスペースである。SQLite の Lumiere（ `db/lumiere.sqlite` ）に資料の所在を記録し、共通語彙で検索と分類をそろえる。Lumiere は資料の本文を持たず、タイトル・所在・用語だけを記録する。外部サービスの資料（Backlog URL、Google Doc URL など）は URL を所在とし、本文はそのサービスから取得する。外部に正本を持たないローカルファイルは `files/` に取り込み、そこを正本とする。

Lumiere のスキーマと CLI の詳細は [docs/lumiere.md](./docs/lumiere.md)、議事録の自動取り込みは [docs/cogsworth.md](./docs/cogsworth.md)、中村宛の依頼の Issue 化は [docs/plumette.md](./docs/plumette.md) にまとめている。

`reference` テーブルが資料の索引である。`query` で絞り込み、本文は `source` から取得する。用語の正本は `terms` テーブル群（共通語彙）で、save 時に title から自動推定する。同一 `source` への再 `save` は upsert される。

| table | role |
| :-- | :-- |
| `reference` | 資料の索引（タイトル・所在） |
| `terms` | 共通語彙（client / meeting / person / project / process / team / system） |
| `term_aliases` | 別名・タイトルマッチ用パターン |
| `reference_terms` | 資料と用語の紐付け |
| `term_relations` | 用語間の関係（works_for, part_of, uses など） |

語彙の学習（ `term learn ID < JSON` ）は、資料の本文から抽出した用語・別名・用語間の関係を登録する。抽出は資料を読んだ Agent が行い、JSON の形と規則は [docs/lumiere.md](./docs/lumiere.md) の Learning に従う。既存の用語は名前と別名で引き当てるため、`阪急` のような別名が新しい用語として増えることはない。会議だけは例外で、語彙の学習では新しく作らない。会議は Google Calendar の予定名で `term add --category meeting` する。

```
batb/
  CLAUDE.md
  docs/lumiere.md
  docs/cogsworth.md
  docs/plumette.md
  schema.sql
  graph.html
  bin/batb
  .claude/scheduled-tasks/
  db/lumiere.sqlite
  files/
  tmp/
```

## CLI reference

よく使う操作を次に示す。`list` は `query` の alias である。

| operation | command |
| :-- | :-- |
| 保存 | `batb save --title T --source URL|PATH [--term NAME ...] [--require-term]` |
| 検索 | `batb query [KEYWORD ...]` / `query --term NAME ...` |
| 用語推定 | `batb term infer "タイトルや文面"` |
| 用語一覧 | `batb term list [--category CAT]` |
| 用語詳細 | `batb term query NAME`（完全一致で詳細表示） |
| 用語追加 | `batb term add --name N --category CAT [--alias A ...]` |
| 用語統合 | `batb term merge SRC --into DST`（別名・紐付け・関係を移す） |
| 用語削除 | `batb term remove NAME`（資料が紐付いていれば拒否する） |
| 語彙学習 | `batb term learn ID < JSON`（本文から抽出した用語と関係を登録する） |
| 資料の用語 | `batb reference link list|add|remove|set ID --term NAME ...` |
| 関係グラフ | `batb graph [--out PATH]` |

## Agent workflow

各ターンは意図、検索、本文、外部補完、整理、回答、保存の 7 段階を回す。同一会話内で既に取得した本文は使い回し、不要な再 fetch を避ける。

```mermaid
flowchart LR
  intent[意図] --> search[検索]
  search --> body[本文]
  body --> external[外部]
  external --> organize[整理]
  organize --> answer[回答]
  answer --> saveStep[保存]
  saveStep --> intent
```

Agent は資料の登録・用語付与・語彙の学習まで行う。本文を読んだ資料は、`save` のあとに本文から用語と関係を抽出し、`term learn ID` に JSON で渡す。

| principle | detail |
| :-- | :-- |
| 先に検索 | 回答・判断・実装の前に reference を検索する |
| 自動保存 | 参照しうる資料と会話で得た新情報は、頼まれなくても `save` する |
| 文脈の再利用 | 同一会話内の取得済み本文を使い回す |
| 不確実性の分離 | 合意・進行中・未確認を混同しない |
| 根拠の明示 | 議事録・課題・予定など、出典を示す |
| 推測の禁止 | reference・Calendar・Backlog を見ずに断定しない |

検索では、クライアント名・会議名・プロジェクト名・課題キー・人名など、文脈から複数パターンを試す。`term infer` で拾える用語を確認してから `query --term` する。論点や決定事項を突き合わせるには、`source` から本文を取得する。

```bash
~/batb/bin/batb term infer "確定：トリプルエスさま定例"
~/batb/bin/batb term query トリプルエス
~/batb/bin/batb query --term トリプルエス
~/batb/bin/batb query 要件 HTML
~/batb/bin/batb query --limit 10
```

## Source formats

`save` するときの `--source` は種別ごとに次の形式で書く。形式をそろえると同一資料の重複登録を防げる。本文は表の fetch 手段で正本から取得する。

ローカルファイルを `--source` に渡すと、`files/` へコピーしたうえで、そのコピーの絶対パスを `source` に記録する。元のファイルが動いたり消えたりしても資料は残る。ファイル名が同じ資料は同じ 1 件として扱われるため、日付や版を含む名前を付ける。

`--require-term` を付けると、title からの用語推定に失敗した場合に save を止める。推定できないときは `term infer` の結果をユーザーに確認し、`--term` で明示してから save する。

| type | source format | fetch |
| :-- | :-- | :-- |
| Backlog | `https://{space}.backlog.com/view/{ISSUE_KEY}` | `get_issue` / `get_issue_comments` |
| Slack | permalink URL | `slack_read_thread` / `slack_read_channel` / `slack_read_file` |
| Google Doc | `https://docs.google.com/.../d/{id}/edit` | `read_file_content` |
| Notion | `https://www.notion.so/{pageId}` | Notion MCP |
| local | `files/{ファイル名}` の絶対パス | ファイル read |

## Cogsworth

Cogsworth は、終了したカレンダー予定に添付された Gemini 議事録を `reference` に登録する仕組みで、Claude デスクトップアプリの定期タスクとして動く。仕組みと運用は [docs/cogsworth.md](./docs/cogsworth.md)、Agent が実行する手順と実行タイミングは `.claude/scheduled-tasks/cogsworth/SKILL.md` にまとめている。

未登録の議事録だけを `save --require-term` で登録し、本文から語彙を学習させて、新規件数を短く報告する。用語を推定できない議事録は推測で登録せず、予定の名前と Google Doc の ID を報告に残す。

## Plumette

Plumette は、議事録と Slack に届いた中村宛の依頼を Linear の Issue にする仕組みで、Claude デスクトップアプリの定期タスクとして動く。仕組みと運用は [docs/plumette.md](./docs/plumette.md)、Agent が実行する手順と実行タイミングは `.claude/scheduled-tasks/plumette/SKILL.md` にまとめている。

未起票の依頼だけを marutto-ops チームに Triage 状態で作り、入口ごとに結果を報告する。重複は直近 `14` 日の自分の Issue と出典を突き合わせて判定するため、Issue の本文は必ず出典の引用から書く。宛先や担当を読み取れない依頼は推測で起票せず、出典の URL を報告に残す。
