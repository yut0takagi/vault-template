# temp-folder プロジェクト設定

Obsidian Vault をベースにした個人ナレッジ＆業務オペレーション基盤のテンプレート。
議事録・日報・ナレッジ・タスク・提案資料・リサーチなどを一元管理し、Claude Code スキルで自動化する。

## ディレクトリ構造

| ディレクトリ | 用途 |
|---|---|
| `議事録/` | 会議の議事録。`{YYYY}/{MM}/` の階層で月別整理 |
| `日報/` | 日次・週次・月次レビュー |
| `ナレッジ/` | 蓄積型の知識ベース（事実・検証済み情報） |
| `思考/` | 持論・仮説・着想（ナレッジ=事実 と分離） |
| `タスク/` | Markwhen(.mw) + Dataview + 詳細md の3層構造 |
| `inbox/` | `temp/`（処理中）→ `まとめ済み/`（ナレッジ化後の原本保管） |
| `アーカイブ/` | 完了・退避ファイル |
| `ログ/` | チャットログ・スキル改善ログ等の自動記録 |
| `添付/` | 議事録の添付素材 |
| `設計/` | Vault自体のアーキテクチャ・テンプレート |
| `制作/` | スライド等の成果物 |
| `Clippings/` | Web Clipper等で保存した記事 |
| `Excalidraw/` | Excalidraw図 |

## ルール（個別ファイル）

詳細は [.claude/rules/](.claude/rules/) を参照。Claude Code が自動ロードする。

- [completion-protocol.md](.claude/rules/completion-protocol.md) — 完了宣言前のチェックリスト
- [no-guessing.md](.claude/rules/no-guessing.md) — 推測禁止、不明点は質問
- [confirm-side-effects.md](.claude/rules/confirm-side-effects.md) — 副作用アクションの事前確認
- [session-quality.md](.claude/rules/session-quality.md) — ループ・スラッシング・再試行の防止
- [chatlog.md](.claude/rules/chatlog.md) — 毎レスポンス末尾でJSONLログ追記
- [minutes-naming.md](.claude/rules/minutes-naming.md) — 議事録の命名規則
- [knowledge-folder-discipline.md](.claude/rules/knowledge-folder-discipline.md) — ナレッジ新規フォルダ禁止
- [knowledge-map-sync.md](.claude/rules/knowledge-map-sync.md) — ナレッジマップ.canvas 更新義務
- [load-user-context.md](.claude/rules/load-user-context.md) — 1on1/提案時の前提情報読み込み
- [task-management.md](.claude/rules/task-management.md) — タスク3層構造の運用
- [csv-encoding.md](.claude/rules/csv-encoding.md) — 日本語CSVはUTF-8 BOM付き
- [external-api-fallback.md](.claude/rules/external-api-fallback.md) — 外部API制限時のフォールバック

## 応答スタイル

- 日本語で応答する
- 成果物はvault相対パスで作成・参照する
- 不要になったファイルは都度削除して整理を保つ

## 初期セットアップ

このテンプレートを使い始めるには `/初期設定` スキルを実行する。MCP サーバー・プラグイン・Obsidian 設定を対話的にセットアップする（[obsidian-second-brain](https://github.com/eugeniughelbur/obsidian-second-brain) の統合も含む）。
