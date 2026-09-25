# Scheduled Tasks

このディレクトリは、Claude デスクトップアプリの定期タスクを Git で管理するために置いている。アプリが読み込むのは `~/.claude/scheduled-tasks/<name>/SKILL.md` だけで、このディレクトリのファイルは読み込まれない。定期タスクの正本はこのディレクトリであり、アプリ側はその写しである。

## Format

各タスクは `<name>/SKILL.md` に置く。本文はアプリに渡すプロンプトで、YAML frontmatter はアプリのタスク設定を記録したものである。

| key | value |
| :-- | :-- |
| `name` | タスク名。ディレクトリ名と揃える |
| `description` | アプリに表示される説明 |
| `schedule` | 実行タイミングの cron 式。アプリはこの値を読まない |
| `timezone` | `schedule` を解釈するタイムゾーン |

## Apply

変更はこのディレクトリで行い、Pull Request で統合してからアプリへ反映する。プロンプトは、アプリが読まない `schedule` と `timezone` を除いて `~/.claude/scheduled-tasks/<name>/SKILL.md` に写す。実行タイミングはアプリのタスク設定で変え、 `schedule` と一致させる。

```shell
grep -v -E '^(schedule|timezone):' .claude/scheduled-tasks/cogsworth/SKILL.md > ~/.claude/scheduled-tasks/cogsworth/SKILL.md
```
