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

#### Google Workspace 系（OAuth 1回で全サービス連携）

**google-workspace** が最も強力。Docs / Drive / Sheets / Slides / Gmail / Calendar / Tasks / Forms / Chat / Contacts を1つのMCPでカバーする統合実装。Google Cloud Console で関連API（Docs / Drive / Sheets / Slides / Gmail / Calendar / Tasks / Forms 等）を有効化したプロジェクトのOAuthクレデンシャル（`credentials.json`）が必要。

| MCP | 用途 | インストールコマンド |
|-----|------|---------------------|
| google-workspace | Docs/Drive/Sheets/Slides/Gmail/Calendar/Tasks/Forms/Chat 統合 | `claude mcp add google_workspace -- uvx workspace-mcp` |
| google-calendar（任意） | カレンダー専用・軽量。workspace と併用可（カレンダーだけ素早く操作したい時用） | `claude mcp add google-calendar -- npx -y @cocal/google-calendar-mcp` |

##### google-workspace のセットアップ手順

1. **Google Cloud Console で API 有効化**（プロジェクト単位）
   - 議事録・ナレッジ用途の最低限: Docs / Drive / Sheets / Slides / Gmail / Calendar / Tasks API
   - フォーム連携も使うなら Forms API、社内チャット連携なら Chat API も追加
2. **OAuth 2.0 クライアントID発行**（デスクトップアプリ種別）→ `credentials.json` をダウンロード
3. **クレデンシャル配置** — 環境変数 `GOOGLE_OAUTH_CREDENTIALS=/path/to/credentials.json` を設定、または `~/.workspace-mcp/credentials.json` に配置
4. **MCP 登録** — `claude mcp add google_workspace -- uvx workspace-mcp`
5. **初回認証** — 任意のツール（例: `mcp__google_workspace__list_calendars`）を呼ぶとブラウザでOAuth同意フローが立ち上がる → `~/.workspace-mcp/tokens/<email>.json` にトークン保存

**重要な注意:**
- OAuth refresh token は通常長期有効だが、**約7日でセッションが切れることがある**（Google側ポリシー・未使用期間・パスワード変更など）。失効時は同じツールを再度呼ぶと再認証フローが走る
- 複数アカウントを切り替える場合は `mcp__google_workspace__manage_accounts` で管理
- `google-calendar` MCP（@cocal/...）と機能が重複する。両方入れても害はないが、用途を分けないなら google-workspace 一本で足りる

#### コミュニケーション系（Claude Code ビルトイン）

| MCP | 用途 | インストールコマンド |
|-----|------|---------------------|
| Slack | Slack連携 | Claude Code のビルトイン機能（プラグイン有効化） |
| Gmail | Gmail連携 | Claude Code のビルトイン機能（自動接続。google-workspace の Gmail と用途で使い分け） |

#### アナリティクス系（任意・データ分析向け）

Google Cloud で BigQuery / Cloud Logging / Cloud Monitoring API などを有効化済みなら、以下を検討。実装が流動的なので**利用前に最新の公式・コミュニティ MCP を確認**すること。

| MCP | 用途 | 備考 |
|-----|------|------|
| bigquery | BigQuery クエリ実行・データセット管理 | 公式実装は流動的。`gcloud auth application-default login` で ADC を通してから利用 |
| cloud-logging | ログ分析 | 公式 MCP が出ていない場合は `gcloud logging read` を Bash 経由で代替 |

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

### Phase 2.5: obsidian-second-brain 統合（強く推奨）

[obsidian-second-brain](https://github.com/eugeniughelbur/obsidian-second-brain) はObsidian vault を「自己書き換え型セカンドブレイン」として運用するための Claude Code スキル群（Karpathy の LLM Wiki パターンを発展させたもの）。**30+ のスラッシュコマンド** が追加され、`/obsidian-daily` `/obsidian-init` `/obsidian-decide` `/research` `/research-deep` `/youtube` `/x-read` `/x-pulse` 等が使えるようになる。

ユーザーに導入するか確認し、Yesならインストール:

```bash
# 1. リポジトリをクローン
git clone https://github.com/eugeniughelbur/obsidian-second-brain.git ~/Develop/obsidian-second-brain

# 2. インストールスクリプト実行（コマンド群を ~/.claude/commands/ にコピー、スキルを ~/.claude/skills/ にリンク）
cd ~/Develop/obsidian-second-brain
bash install.sh

# 3. (任意) Research toolkit を使うなら API キー設定
# install.sh が対話的に聞いてくる: Grok (xAI), Perplexity, YouTube API
# 後から手動設定する場合: ~/.config/obsidian-second-brain/.env を編集

# 4. vault パスを環境変数に設定（オプションだが推奨）
echo 'export OBSIDIAN_VAULT_PATH="$HOME/vault"' >> ~/.zshrc  # 自分のvaultパスに置き換える
```

**インストール後の動作確認:**

```bash
# Claude Code を再起動して、コマンド一覧に obsidian-* が追加されているか確認
ls ~/.claude/commands/ | grep obsidian
ls ~/.claude/skills/ | grep obsidian-second-brain
```

**追加されるコマンドの一部:**

| コマンド | 用途 |
|---|---|
| `/obsidian-daily` | デイリーノート自動生成（カレンダー・タスク・会話文脈を反映） |
| `/obsidian-init` | vault 初期化（プリセット選択: personal/team/research/builder） |
| `/obsidian-decide` `/obsidian-adr` | 意思決定の記録（ADR形式） |
| `/obsidian-health` | vault 健全性チェック（壊れたリンク・古い情報・矛盾検出） |
| `/research` `/research-deep` | Web リサーチ → vault に AI-first ノートとして保存 |
| `/x-read` `/x-pulse` | X (Twitter) 投稿の深掘り読解 |
| `/youtube` | YouTube動画の文字起こし → ノート化 |
| `/notebooklm` | NotebookLM 経由のソース固定リサーチ |

導入後、`CLAUDE.md` の構造と整合させるため `_CLAUDE.md` の運用ルールを確認すると良い。

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
mcp__google_workspace__*
mcp__google-calendar__*
mcp__slack__*
mcp__claude_ai_Gmail__*
mcp__claude_ai_Google_Drive__*
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
- google-workspace: `mcp__google_workspace__list_calendars` を呼んでブラウザOAuth同意 → カレンダー一覧が返ってくれば成立。続けて `mcp__google_workspace__list_drive_items`（Drive）, `mcp__google_workspace__search_gmail_messages`（Gmail）も1件ずつ叩いて疎通確認
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
- **google-workspace / google-calendar も7日でトークン切れの可能性**。初回起動時にOAuth認証フローが走る。`invalid_grant` エラーが出たら同じツールをもう一度呼んで再認証フローを走らせる
- google-workspace を使うなら、Google Cloud Console で**必要なAPIをすべて有効化済みであること**が前提（Docs/Drive/Sheets/Slides/Gmail/Calendar/Tasks など）。未有効化のAPIを叩くと403が返る
