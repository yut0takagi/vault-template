---
name: GASコード作成
description: "Google Apps Script（GAS）のコードを作成・修正。スプレッドシート操作・Slack連携・フォーム連携・外部API呼び出し・トリガー設定に対応。Use when: 「GAS」「Google Apps Script」「スプシ連携」「Slack投稿」「フォーム連携」「GASで自動化」"
argument-hint: "[作りたいGASの説明]"
allowed-tools: "Read, Write, Edit, Glob, Grep, Agent"
---

$ARGUMENTS の説明に基づいて、Google Apps Script（GAS）のコードを生成してください。

引数（必須）: 作りたいGASの説明
（例: `/GASコード作成 Googleフォームの回答をSlackに通知`）
（例: `/GASコード作成 スプレッドシートの日次データをまとめてSlack投稿`）
（例: `/GASコード作成 外部APIからデータ取得してスプシに書き込み`）

---

## ステップ0: リファレンス読み込み

GAS書き方ガイドのナレッジファイルがプロジェクト内にあれば読み込む。

---

## ステップ1: 要件分析

ユーザーの説明から以下を分析する：

### 1.1 モード判定

| モード | 判定条件 |
|--------|---------|
| スプシ操作 | 「読み込み」「書き込み」「集計」「フィルタ」「書式」 |
| Slack連携 | 「Slack」「通知」「投稿」「日報」 |
| フォーム連携 | 「フォーム」「回答」「送信時」 |
| 外部API | 「API」「取得」「連携」「Webhook」 |
| 定期実行 | 「毎日」「毎時」「定期」「自動」 |
| 双方向 | 「Webアプリ」「受信」「Slash Command」 |

### 1.2 設計案の提示

```
## GAS設計

**目的:** {やりたいこと}
**トリガー:** {手動実行 / 時間トリガー / フォーム送信 / 編集時 / Webアプリ}
**入力:** {スプシ / フォーム / API / Slackコマンド}
**処理:** {処理ステップの概要}
**出力:** {スプシ書き込み / Slack投稿 / API応答 / メール送信}

### 必要なサービス
- SpreadsheetApp / FormApp / GmailApp / UrlFetchApp 等

この設計でよろしいですか？
```

**ユーザーの承認を待つ**

---

## ステップ2: コード生成

### 2.1 鉄則（必ず守ること）

**パフォーマンス:**
- スプシ操作は `getValues()` / `setValues()` で一括処理。1セルずつのループは厳禁（70倍遅い）
- `getDataRange()` でデータ範囲を自動検出、メモリ上で加工してから書き戻す

**セキュリティ:**
- APIキー・トークンは `PropertiesService.getScriptProperties()` に保存。コード内直書き厳禁
- スプシのセルにトークンを書くのもNG

**V8構文:**
- `const` / `let` を使う（`var` 非推奨）
- アロー関数、分割代入、テンプレートリテラル、Optional Chaining(`?.`)、Nullish Coalescing(`??`) を活用
- `import`/`export` / `setTimeout` / `fetch` API / `#privateField` は使えない

**エラーハンドリング:**
- 外部API呼び出しには必ず `muteHttpExceptions: true` を付ける
- `try-catch` で例外を捕捉し、`console.error()` でログ出力
- 重要な処理はSlackやメールでエラー通知

**制限値の意識:**
- 実行時間: 6分/回
- URLフェッチ: 無料20,000回/日、Workspace 100,000回/日
- トリガー: 20個/ユーザー/スクリプト

### 2.2 コードテンプレート

#### スプシ一括操作

```javascript
function processSheet() {
  const sheet = SpreadsheetApp.getActiveSpreadsheet().getSheetByName('シート名');
  const [header, ...rows] = sheet.getDataRange().getValues();
  
  const updated = rows.map(row => {
    // 加工処理
    return row;
  });
  
  const output = [header, ...updated];
  sheet.getDataRange().setValues(output);
}
```

#### Slack投稿（Incoming Webhook）

```javascript
function sendToSlack(message) {
  const webhookUrl = PropertiesService.getScriptProperties().getProperty('SLACK_WEBHOOK_URL');
  
  UrlFetchApp.fetch(webhookUrl, {
    method: 'post',
    contentType: 'application/json',
    payload: JSON.stringify({ text: message }),
    muteHttpExceptions: true
  });
}
```

#### Slack投稿（Block Kit リッチメッセージ）

```javascript
function sendRichSlack(title, fields, body) {
  const webhookUrl = PropertiesService.getScriptProperties().getProperty('SLACK_WEBHOOK_URL');
  
  const payload = {
    blocks: [
      { type: 'header', text: { type: 'plain_text', text: title } },
      {
        type: 'section',
        fields: fields.map(f => ({ type: 'mrkdwn', text: f }))
      },
      { type: 'section', text: { type: 'mrkdwn', text: body } },
      { type: 'divider' },
      { type: 'context', elements: [{ type: 'mrkdwn', text: '自動投稿 by GAS' }] }
    ]
  };
  
  UrlFetchApp.fetch(webhookUrl, {
    method: 'post',
    contentType: 'application/json',
    payload: JSON.stringify(payload),
    muteHttpExceptions: true
  });
}
```

#### Slack Bot Token（chat.postMessage）

```javascript
function postToChannel(channel, text, blocks) {
  const token = PropertiesService.getScriptProperties().getProperty('SLACK_BOT_TOKEN');
  
  const response = UrlFetchApp.fetch('https://slack.com/api/chat.postMessage', {
    method: 'post',
    headers: { 'Authorization': `Bearer ${token}` },
    contentType: 'application/json',
    payload: JSON.stringify({ channel, text, blocks }),
    muteHttpExceptions: true
  });
  
  const result = JSON.parse(response.getContentText());
  if (!result.ok) console.error(`Slack Error: ${result.error}`);
  return result;
}
```

#### 外部API呼び出し（リトライ付き）

```javascript
function fetchWithRetry(url, options = {}, maxRetries = 3) {
  const defaults = { method: 'get', muteHttpExceptions: true };
  const opts = { ...defaults, ...options };
  
  for (let attempt = 1; attempt <= maxRetries; attempt++) {
    try {
      const response = UrlFetchApp.fetch(url, opts);
      const code = response.getResponseCode();
      if (code === 200) return JSON.parse(response.getContentText());
      if ((code === 429 || code >= 500) && attempt < maxRetries) {
        Utilities.sleep(attempt * 2000);
        continue;
      }
      throw new Error(`HTTP ${code}: ${response.getContentText()}`);
    } catch (e) {
      if (attempt === maxRetries) throw e;
      Utilities.sleep(attempt * 2000);
    }
  }
}
```

#### フォーム送信トリガー

```javascript
function onFormSubmit(e) {
  const responses = e.namedValues;
  const sheet = e.range.getSheet();
  const row = e.range.getRow();
  
  // 回答データを取得
  const name = responses['名前'][0];
  const email = responses['メール'][0];
  
  // 後処理（ステータス列追加、通知等）
  sheet.getRange(row, sheet.getLastColumn() + 1).setValue('未対応');
  sendToSlack(`新しい回答: ${name} (${email})`);
}
```

#### 時間トリガー設定

```javascript
function createTimeTrigger() {
  // 毎日17時（JST）
  ScriptApp.newTrigger('dailyTask')
    .timeBased()
    .atHour(17)
    .everyDays(1)
    .inTimezone('Asia/Tokyo')
    .create();
}

// トリガー全削除（再設定時に使用）
function deleteAllTriggers() {
  ScriptApp.getProjectTriggers().forEach(trigger => {
    ScriptApp.deleteTrigger(trigger);
  });
}
```

#### Webアプリ（doPost / doGet）

```javascript
// デプロイ: ウェブアプリ → アクセス: 全員
function doPost(e) {
  const body = JSON.parse(e.postData.contents);
  
  // Slack Events API: URL検証
  if (body.type === 'url_verification') {
    return ContentService.createTextOutput(body.challenge);
  }
  
  // 処理
  const result = processRequest(body);
  
  return ContentService.createTextOutput(JSON.stringify(result))
    .setMimeType(ContentService.MimeType.JSON);
}

function doGet(e) {
  const params = e.parameter;
  return ContentService.createTextOutput(JSON.stringify({ status: 'ok' }))
    .setMimeType(ContentService.MimeType.JSON);
}
```

#### スプシDB（CRUDパターン）

```javascript
class SheetDB {
  constructor(sheetName) {
    this.sheet = SpreadsheetApp.getActiveSpreadsheet().getSheetByName(sheetName);
  }

  findAll() {
    const [header, ...rows] = this.sheet.getDataRange().getValues();
    return rows.map(row => {
      const obj = {};
      header.forEach((key, i) => obj[key] = row[i]);
      return obj;
    });
  }

  findBy(column, value) {
    return this.findAll().filter(row => row[column] === value);
  }

  insert(obj) {
    const header = this.sheet.getRange(1, 1, 1, this.sheet.getLastColumn()).getValues()[0];
    this.sheet.appendRow(header.map(key => obj[key] ?? ''));
  }

  update(keyColumn, keyValue, updates) {
    const data = this.sheet.getDataRange().getValues();
    const header = data[0];
    const keyIdx = header.indexOf(keyColumn);
    for (let i = 1; i < data.length; i++) {
      if (data[i][keyIdx] === keyValue) {
        for (const [col, val] of Object.entries(updates)) {
          const colIdx = header.indexOf(col);
          if (colIdx >= 0) data[i][colIdx] = val;
        }
      }
    }
    this.sheet.getDataRange().setValues(data);
  }

  delete(keyColumn, keyValue) {
    const data = this.sheet.getDataRange().getValues();
    const keyIdx = data[0].indexOf(keyColumn);
    for (let i = data.length - 1; i >= 1; i--) {
      if (data[i][keyIdx] === keyValue) this.sheet.deleteRow(i + 1);
    }
  }
}
```

#### 大量データ処理（6分の壁対策）

```javascript
function processLargeData() {
  const props = PropertiesService.getScriptProperties();
  const startRow = parseInt(props.getProperty('PROGRESS_ROW') || '2');
  const BATCH_SIZE = 500;
  
  const sheet = SpreadsheetApp.getActiveSheet();
  const lastRow = sheet.getLastRow();
  const endRow = Math.min(startRow + BATCH_SIZE - 1, lastRow);
  
  if (startRow > lastRow) {
    props.deleteProperty('PROGRESS_ROW');
    sendToSlack('処理完了');
    return;
  }
  
  const range = sheet.getRange(startRow, 1, endRow - startRow + 1, sheet.getLastColumn());
  const values = range.getValues();
  const processed = values.map(row => { /* 処理 */ return row; });
  range.setValues(processed);
  
  if (endRow < lastRow) {
    props.setProperty('PROGRESS_ROW', String(endRow + 1));
    ScriptApp.newTrigger('processLargeData').timeBased().after(1000).create();
  } else {
    props.deleteProperty('PROGRESS_ROW');
    sendToSlack('処理完了');
  }
}
```

#### PropertiesService 初期設定ヘルパー

```javascript
// 初回セットアップ用（手動実行）
function setupProperties() {
  const props = PropertiesService.getScriptProperties();
  props.setProperties({
    'SLACK_WEBHOOK_URL': 'https://hooks.slack.com/services/xxx/yyy/zzz',
    'SLACK_BOT_TOKEN': 'xoxb-xxxx',
    'API_KEY': 'your-api-key-here',
  });
  console.log('Properties set:', Object.keys(props.getProperties()));
}
```

---

## ステップ3: コード出力

### 3.1 出力形式

生成したコードを以下のいずれかで出力：

| 出力先 | 条件 |
|--------|------|
| チャットに表示 | 短いコード（〜100行）or ユーザーが手動コピペする場合 |
| `.js` ファイル | 長いコード or 複数ファイル構成の場合 |

ファイル出力する場合:
- 保存先: プロジェクト内の `inbox/gas/` ディレクトリ
- ファイル名: `{機能名（スネークケース）}.js`
- ディレクトリがなければ作成

### 3.2 付属ドキュメント

コードと一緒に以下を出力する：

```
## GAS コード作成完了

**ファイル:** {パス or 「以下のコード」}
**トリガー:** {手動 / 時間 / フォーム送信 / 編集時 / Webアプリ}

### セットアップ手順
1. [Google Apps Script](https://script.google.com) で新規プロジェクト作成
   （スプシ連動の場合: スプシ → 拡張機能 → Apps Script）
2. コードを貼り付け
3. `setupProperties()` を実行してAPIキー/トークンを設定
4. {トリガー設定の説明}
5. {権限の承認}

### 使用するプロパティ
| キー | 値 | 取得方法 |
|------|-----|---------|
| {KEY} | {説明} | {取得手順} |

### 注意事項
- {制限値・クォータ等}
```

---

## ステップ4: セルフチェック

生成したコードを以下で検証：

- [ ] `getValues()` / `setValues()` で一括処理している（1セルずつのループがない）
- [ ] APIキー・トークンが `PropertiesService` で管理されている（ハードコードなし）
- [ ] 外部API呼び出しに `muteHttpExceptions: true` が付いている
- [ ] `var` が使われていない（`const` / `let` のみ）
- [ ] トリガーから呼ぶ関数が通常の `function` 宣言になっている（アロー関数でない）
- [ ] 6分の実行時間制限内で完了する設計になっている
- [ ] エラー時の通知・ログ出力がある
- [ ] `deleteRow` のループが下から実行されている（行ズレ防止）

---

## ステップ5: ナレッジ参照パスの検証

コード生成前にリファレンスファイルの存在を確認する：
- GAS書き方ガイドのナレッジファイルが存在するかGlobで確認
- 存在しない場合は `/リサーチ GAS` の実行を提案

---

## ステップ6: デバッグ・デプロイガイド

コード出力時に以下も併記する：

```
### デバッグ手順
1. GASエディタで対象関数を選択 → ▶ 実行
2. 「実行ログ」パネルで `console.log` / `console.error` の出力を確認
3. エラー発生時はスタックトレースの行番号を確認

### clasp を使う場合（ローカル開発）
1. `npm install -g @google/clasp`
2. `clasp login` → Google認証
3. `clasp clone {スクリプトID}` → ローカルにソース取得
4. ローカルで編集 → `clasp push` でアップロード
5. `clasp open` でブラウザのGASエディタを開く

### よくあるエラーと対処
| エラー | 原因 | 対処 |
|--------|------|------|
| `TypeError: Cannot read properties of null` | シート名の不一致 | `getSheetByName()` の引数を確認 |
| `Exception: Request failed for https://...` | API側のエラー | レスポンスコードとbodyをログ出力して確認 |
| `Exceeded maximum execution time` | 6分超過 | バッチ処理に分割（大量データ処理テンプレート参照） |
| `You do not have permission` | OAuth スコープ不足 | appsscript.json に必要なスコープを追加 |
```
