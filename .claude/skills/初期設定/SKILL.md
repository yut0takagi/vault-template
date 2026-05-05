---
name: 初期設定
description: MCP サーバー・プラグイン・Obsidian 設定を対話的にセットアップし、vault の全機能を使えるようにする
argument-hint: (引数なし)
allowed-tools: Bash, Read, Write, Edit, WebSearch, WebFetch
---

# 初期設定スキル

このVaultテンプレートの全機能を使えるようにするための初期セットアップウィザード。  
対話的に進め、ユーザーの環境に合わせて必要なものだけインストールする。

---

## 実行フロー

### Phase 0: 環境チェック

まず現在の環境を確認する。

```bash
# 必須ツール確認
echo "=== 環境チェック ==="
echo "Node.js: $(node --version 2>/dev/null || echo '未インストール')"
echo "npm: $(npm --version 2>/dev/null || echo '未インストール')"
echo "npx: $(npx --version 2>/dev/null || echo '未インストール')"
echo "Python: $(python3 --version 2>/dev/null || echo '未インストール')"
echo "Git: $(git --version 2>/dev/null || echo '未インストール')"
echo "Claude Code: $(claude --version 2>/dev/null || echo '未インストール')"
echo "Homebrew: $(brew --version 2>/dev/null | head -1 || echo '未インストール')"
```

```bash
# 既存MCP確認
claude mcp list 2>/dev/null || echo "MCP一覧取得不可"
```

結果をユーザーに見せ、不足があれば案内する。  
Node.js が無い場合は `brew install node` を提案。

---

### Phase 1: MCP サーバーセットアップ

以下のMCPサーバーを**カテゴリ別に提案**し、ユーザーが選択したものをインストールする。

#### コア（強く推奨）

| MCP | 用途 | インストールコマンド |
|-----|------|---------------------|
| filesystem | ファイル操作 | `claude mcp add filesystem -- npx -y @modelcontextprotocol/server-filesystem $HOME` |
| github | GitHub連携 | `claude mcp add github -- npx -y @modelcontextprotocol/server-github` |
| playwright | ブラウザ自動操作 | `claude mcp add playwright -- npx -y @playwright/mcp@latest` |
| memory | 永続メモリ | `claude mcp add memory -- npx -y @modelcontextprotocol/server-memory` |

#### 情報収集・リサーチ系

| MCP | 用途 | インストールコマンド |
|-----|------|---------------------|
| whisper | 音声文字起こし | `claude mcp add whisper -- npx -y whisper-mcp` |
| notebooklm-mcp | NotebookLM連携 | （3ステップ・下記参照） |

##### NotebookLM のセットアップ（重要：パッケージ名と順序に注意）

⚠️ **パッケージ名の罠**: PyPI には `notebooklm-mcp` と `notebooklm-mcp-cli` という似た名前の **全く別物のパッケージ** が存在する。
このテンプレが想定するのは **`notebooklm-mcp-cli`** の方（`nlm` CLI を同梱、Google アカウントでブラウザログインする方式）。
誤って `notebooklm-mcp`（Selenium ベース・別チーム）を入れると `nlm` コマンドが存在せず、`notebooklm-mcp init <URL>` を要求されて詰むので注意。

NotebookLM は **インストール → ログイン → 検証 → MCP登録** の順で進める。
ログイン前に MCP を登録すると、初回呼び出しで必ず失敗する。

```bash
# ① CLI インストール（uv 推奨。nlm と notebooklm-mcp の2バイナリが入る）
uv tool install notebooklm-mcp-cli
# uv が無い環境では: pip3 install --user notebooklm-mcp-cli

# ② Google アカウントでログイン（ブラウザが立ち上がる）
nlm login

# ③ ログイン状態を確認（"Cookies: present" / "Account: ..." が出ればOK）
nlm doctor

# ④ MCP として Claude Code に登録
claude mcp add notebooklm-mcp -- notebooklm-mcp
```

**重要な注意:**
- Cookie ベース認証なので **約7日でセッションが切れる**。`nlm login --check` で検査でき、切れていたら `nlm login` で再認証
- 複数 Google アカウントを使い分ける場合は `nlm login switch <profile>` でデフォルトを切り替える
- Gemini CLI / Cursor / Windsurf を使っているなら `nlm setup add <client>` で MCP を自動登録できる（Claude Code は対象外なので手動の `claude mcp add` が必要）

#### コミュニケーション系

| MCP | 用途 | インストールコマンド |
|-----|------|---------------------|
| google-calendar | カレンダー連携 | `claude mcp add google-calendar -- npx -y @cocal/google-calendar-mcp` |
| Slack | Slack連携 | Claude Code のビルトイン機能（プラグイン有効化） |
| Gmail | Gmail連携 | Claude Code のビルトイン機能（プラグイン有効化） |

#### 開発支援系

| MCP | 用途 | インストールコマンド |
|-----|------|---------------------|
| context7 | ライブラリドキュメント検索 | `claude mcp add context7 -- npx -y @upstash/context7-mcp` |
| sequential-thinking | 段階的思考 | `claude mcp add sequential-thinking -- npx -y @modelcontextprotocol/server-sequential-thinking` |

**進め方:**
1. カテゴリごとに「インストールしますか？」と確認
2. 「全部入れる」も選択肢として提示
3. 各インストール後に `claude mcp list` で接続確認
4. 失敗したものはスキップして最後にまとめて報告

---

### Phase 2: プラグイン設定

```bash
# Slack プラグイン有効化（ビルトイン）
# settings.json の enabledPlugins に追加
```

ユーザーに確認して、`settings.json` の `enabledPlugins` に追加する：

```json
{
  "enabledPlugins": {
    "slack@claude-plugins-official": true
  }
}
```

---

### Phase 3: settings.json 権限更新

インストールしたMCPに合わせて、`.claude/settings.json` の `permissions.allow` にMCP権限を追加する。

**インストールしたMCPのみ** 権限を追加する（未インストールのは追加しない）：

```
mcp__filesystem__*
mcp__github__*
mcp__playwright__*
mcp__memory__*
mcp__whisper__*
mcp__notebooklm-mcp__*
mcp__google-calendar__*
mcp__slack__*
mcp__claude_ai_Gmail__*
```

---

### Phase 4: Obsidian 設定確認

1. `.obsidian/app.json` が存在するか確認
2. 推奨プラグインを案内（手動インストール）:

| プラグイン | 用途 |
|-----------|------|
| Templater | テンプレートから日報・月報を自動生成 |
| Dataview | ナレッジ・タスクの動的クエリ |
| Tasks | チェックボックスタスク管理 |
| Excalidraw | 図・ホワイトボード |
| Git | 自動バックアップ |
| Calendar | カレンダービュー |

**注意:** Obsidianプラグインはコマンドラインからインストールできないため、リストを提示するだけ。

---

### Phase 5: 動作確認

設定完了後、実際にスキルが動くかテストする。

```bash
# ファイル操作テスト
echo "テスト" > inbox/temp/setup-test.md && cat inbox/temp/setup-test.md && rm inbox/temp/setup-test.md
echo "✓ ファイル操作OK"
```

各MCPの簡易テスト：
- filesystem: ディレクトリ一覧取得
- github: 自分のリポジトリ一覧
- playwright: ページスナップショット
- google-calendar: カレンダー一覧
- notebooklm-mcp: `nlm doctor` で `Cookies: present` と `Account:` が表示されること、`nlm login --check` で `Auth is valid` が出ること。両方OKなら `mcp__notebooklm-mcp__notebook_list` を1回叩いて応答を確認

---

### Phase 6: 完了レポート

最後に以下をまとめて報告：

```
=== セットアップ完了 ===

✓ インストール済みMCP:
  - filesystem
  - github
  - playwright
  - ...

✗ 未インストール（スキップ）:
  - whisper（不要と判断）
  - ...

✓ 権限設定: .claude/settings.json 更新済み
✓ Obsidian設定: .obsidian/app.json 確認済み

推奨プラグイン（Obsidianで手動インストール）:
  - Templater, Dataview, Tasks, Excalidraw, Git, Calendar

次のステップ:
  1. Obsidianでこのフォルダを開く
  2. 推奨プラグインをインストール
  3. `/リサーチ` や `/深掘り` でスキルを試す
```

---

## 注意事項

- グローバルMCP（`~/.claude/` 配下）は変更しない。プロジェクトローカルの `.mcp.json` に追加する
- 既にインストール済みのMCPは再インストールしない
- API キーが必要なサービスは、`.env` ファイルに保存するよう案内する
- **NotebookLM の認証は約7日で切れる**。`/議事録作成` 等を実行する前に `nlm login --check` で検証し、`Auth is invalid` なら `nlm login` で再認証するようユーザーに案内する。MCP登録より先にログインしておかないと初回呼び出しが必ず失敗する
- Google Calendar も同様に7日でトークン切れの可能性。初回起動時にOAuth認証フローが走る
