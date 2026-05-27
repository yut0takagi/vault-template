# .claude/rules/ — 行動ルール集

このディレクトリのMarkdownファイルは、Claude Code がvault作業中に **常時参照** する行動ルール。`CLAUDE.md` から相対パスで列挙されており、自動ロードされる。

## 収録ルール

| ファイル | 概要 |
|---|---|
| [completion-protocol.md](completion-protocol.md) | 完了宣言前のチェックリスト |
| [no-guessing.md](no-guessing.md) | 不明点は推測せず質問 |
| [confirm-side-effects.md](confirm-side-effects.md) | 副作用アクションは事前確認 |
| [session-quality.md](session-quality.md) | ループ・スラッシング・再試行を防ぐ |
| [chatlog.md](chatlog.md) | チャットログJSONLの自動記録 |
| [minutes-naming.md](minutes-naming.md) | 議事録の命名規則 |
| [knowledge-folder-discipline.md](knowledge-folder-discipline.md) | ナレッジ新規フォルダ禁止 |
| [knowledge-map-sync.md](knowledge-map-sync.md) | ナレッジマップ.canvas 同期義務 |
| [load-user-context.md](load-user-context.md) | 1on1/提案時の前提情報読み込み |
| [task-management.md](task-management.md) | タスク3層構造の運用 |
| [csv-encoding.md](csv-encoding.md) | 日本語CSVはUTF-8 BOM付き |
| [external-api-fallback.md](external-api-fallback.md) | 外部API制限時のフォールバック |

## カスタマイズ

- 環境固有のルール（社用アカウント・特定DBのコスト管理など）は、このディレクトリ内に別ファイルで追加する
- 各ルールはfrontmatterに `description:` を持ち、サブagentがロードする際の手がかりにする
- ルール同士の参照は wikilink (`[[ルール名]]`) で書く
