# Claude Code テンプレート

Claude Code プロジェクトの初期設定テンプレート。

## フォルダ構成

```
your-project/
├── CLAUDE.md                          # プロジェクト指示書（最重要）
├── .gitignore
└── .claude/
    ├── settings.json                  # 権限・プラグイン設定
    ├── commands/                      # カスタムスラッシュコマンド
    │   └── sample.md                  # /sample で呼び出せる
    └── skills/                        # スキル（再利用可能な手順書）
        └── サンプルスキル/
            └── SKILL.md
```

## 各ファイルの役割

### `CLAUDE.md`（必須）
プロジェクト全体のルールを定義するファイル。Claude Codeが毎回読み込む。
- 応答言語・スタイル
- コーディング規約
- Git運用ルール
- プロジェクト固有の注意事項

### `.claude/settings.json`
権限の許可・拒否を設定する。
- `allow`: 自動許可するツール呼び出し
- `deny`: 明示的に拒否するツール呼び出し

### `.claude/commands/`
`/コマンド名` で呼び出せるカスタムコマンド。
- ファイル名がコマンド名になる（`sample.md` → `/sample`）
- `$ARGUMENTS` で引数を受け取れる

### `.claude/skills/`
スキルは再利用可能な手順書。`スキル名/SKILL.md` の構造で配置する。

## 使い方

1. このテンプレートをプロジェクトルートにコピー
2. `CLAUDE.md` の `{PROJECT_NAME}` 等を自分のプロジェクトに合わせて編集
3. `.claude/settings.json` の権限を調整
4. 必要に応じてスキルやコマンドを追加

## よくある追加設定

### チーム開発の場合
`CLAUDE.md` と `.claude/commands/` はGitで共有し、
`settings.json` は `.gitignore` に入れて個人設定にする。

### よく使うスキルの例
- `日報作成/SKILL.md` — 日報のフォーマットと手順
- `コードレビュー/SKILL.md` — レビュー観点チェックリスト
- `リリース準備/SKILL.md` — リリース前の確認手順
