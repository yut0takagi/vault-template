---
name: Webテスト
description: "Playwrightを使ってWebアプリケーションのテスト・操作・スクリーンショット取得を実行。Use when: 「Webテスト」「ブラウザテスト」「Playwright」「画面テスト」「E2Eテスト」"
argument-hint: "[テスト内容 or URL]"
---

Playwrightを使ってWebアプリケーションのテスト・操作を行ってください。

引数: $ARGUMENTS
（例: `/Webテスト localhost:3000 のフォーム入力をテスト` `/Webテスト このHTMLの表示確認` `/Webテスト スクリーンショットを撮って`）

---

## 依存パッケージ

```bash
pip install playwright
python -m playwright install chromium
```

---

## アプローチの決定

```
ユーザーのタスク → 静的HTML？
    はい → HTMLファイルを直接読み、セレクタを特定 → Playwrightスクリプト作成
    いいえ（動的Webアプリ）→ サーバーは起動済み？
        いいえ → サーバー起動コマンドを確認して起動
        はい → 偵察→操作パターンで実行
```

---

## 基本テンプレート

```python
from playwright.sync_api import sync_playwright

with sync_playwright() as p:
    browser = p.chromium.launch(headless=True)
    page = browser.new_page()
    page.goto('http://localhost:3000')
    page.wait_for_load_state('networkidle')  # 重要: JSの実行完了を待つ

    # --- ここに操作を記述 ---

    browser.close()
```

---

## 偵察→操作パターン（動的アプリの基本）

### 1. 偵察: ページの状態を把握

```python
# スクリーンショットで視覚確認
page.screenshot(path='/tmp/inspect.png', full_page=True)

# DOM構造を確認
content = page.content()

# 要素の一覧取得
buttons = page.locator('button').all()
links = page.locator('a').all()
inputs = page.locator('input').all()

for btn in buttons:
    print(f"Button: {btn.text_content()} | visible: {btn.is_visible()}")
```

### 2. セレクタを特定

偵察結果からセレクタを決定する：
- `text=ログイン` — テキストで特定
- `role=button[name="送信"]` — ロールで特定
- `#submit-btn` — ID
- `.form-input` — CSSクラス

### 3. 操作を実行

```python
# クリック
page.click('text=ログイン')

# テキスト入力
page.fill('#email', 'test@example.com')
page.fill('#password', 'password123')

# セレクトボックス
page.select_option('#category', 'option_value')

# 待機
page.wait_for_selector('.result', state='visible')

# スクリーンショット（操作後の確認）
page.screenshot(path='/tmp/after_action.png')
```

---

## よくある操作パターン

### フォーム入力とサブミット

```python
page.fill('#name', '鈴木太郎')
page.fill('#email', 'taro@example.com')
page.click('button[type="submit"]')
page.wait_for_load_state('networkidle')
page.screenshot(path='/tmp/form_result.png')
```

### コンソールログの取得

```python
logs = []
page.on('console', lambda msg: logs.append(f"{msg.type}: {msg.text}"))

page.goto('http://localhost:3000')
page.wait_for_load_state('networkidle')

for log in logs:
    print(log)
```

### 静的HTMLファイルのテスト

```python
page.goto('file:///path/to/test.html')
# networkidle は不要（ローカルファイル）
```

### 複数ページの巡回

```python
urls = ['/page1', '/page2', '/page3']
for url in urls:
    page.goto(f'http://localhost:3000{url}')
    page.wait_for_load_state('networkidle')
    page.screenshot(path=f'/tmp/screenshot_{url.replace("/","_")}.png')
```

---

## サーバー起動が必要な場合

```python
import subprocess
import time

# サーバーをバックグラウンドで起動
server = subprocess.Popen(
    ['npm', 'run', 'dev'],
    cwd='/path/to/project',
    stdout=subprocess.PIPE,
    stderr=subprocess.PIPE
)

time.sleep(5)  # サーバー起動待ち

try:
    # Playwrightでテスト実行
    with sync_playwright() as p:
        browser = p.chromium.launch(headless=True)
        page = browser.new_page()
        page.goto('http://localhost:3000')
        page.wait_for_load_state('networkidle')
        # テスト実行...
        browser.close()
finally:
    server.terminate()
    server.wait()
```

---

## 重要な注意事項

1. **`networkidle` を待つ**: 動的アプリでは DOM 検査の前に必ず `page.wait_for_load_state('networkidle')` を実行する。これを忘れるとJSが未実行の状態でDOMを読んでしまう
2. **`headless=True`**: 常にヘッドレスモードで起動する
3. **ブラウザを閉じる**: `browser.close()` を忘れない
4. **スクリーンショットで確認**: 操作の前後でスクリーンショットを撮り、Read ツールで画像を確認する
5. **セレクタの優先順位**: text > role > ID > CSSクラス > XPath（可読性順）

---

## 結果の報告

テスト完了後、以下を報告する：
- テスト対象URL
- 実行した操作の概要
- スクリーンショットのパス
- 発見した問題点（あれば）
