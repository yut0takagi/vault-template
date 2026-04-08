---
name: スプレッドシート
description: "Excelスプレッドシート(.xlsx)の作成・編集・分析。データ整形・集計・グラフ作成にも対応。Use when: 「Excel」「xlsx」「スプレッドシート」「集計」「表を作って」"
argument-hint: "[テーマ or ファイルパス]"
---

以下の手順でExcelスプレッドシート（.xlsx）の作成・編集・分析を行ってください。

引数: $ARGUMENTS
（例: `/スプレッドシート 売上データを整理して` `/スプレッドシート KPIダッシュボードを作って` `/スプレッドシート このCSVをExcelに変換して`）

---

## モード判定

| モード | 判定条件 |
|--------|---------|
| 新規作成 | 「作成」「作って」「ダッシュボード」「テンプレート」 |
| 既存編集 | `.xlsx`/`.csv`/`.tsv` パス指定、「編集」「列追加」「修正」 |
| データ分析 | 「分析」「集計」「統計」「可視化」「グラフ」 |
| 変換 | 「変換」「CSVから」「整形」「クリーニング」 |

---

## ツール選択の基準

| ツール | 用途 |
|--------|------|
| **pandas** | データ分析、一括操作、シンプルなデータ出力 |
| **openpyxl** | 複雑な書式設定、数式、Excel固有の機能 |

---

## 鉄則: 数式を使う（値のハードコードNG）

```python
# NG: Pythonで計算してハードコード
total = df['Sales'].sum()
sheet['B10'] = total  # 5000がハードコードされる

# OK: Excel数式を使う
sheet['B10'] = '=SUM(B2:B9)'
sheet['C5'] = '=(C4-C2)/C2'
sheet['D20'] = '=AVERAGE(D2:D19)'
```

スプレッドシートはデータ変更時に自動再計算できることが重要。

---

## 新規作成

```python
from openpyxl import Workbook
from openpyxl.styles import Font, PatternFill, Alignment, Border, Side

wb = Workbook()
sheet = wb.active
sheet.title = "データ"

# ヘッダー
headers = ["項目", "値", "備考"]
header_font = Font(bold=True, color="FFFFFF", name="Arial")
header_fill = PatternFill('solid', start_color='4472C4')

for col, header in enumerate(headers, 1):
    cell = sheet.cell(row=1, column=col, value=header)
    cell.font = header_font
    cell.fill = header_fill
    cell.alignment = Alignment(horizontal='center')

# データ入力
data = [
    ["売上", 1000000, "1月分"],
    ["コスト", 600000, "1月分"],
]
for row_idx, row_data in enumerate(data, 2):
    for col_idx, value in enumerate(row_data, 1):
        sheet.cell(row=row_idx, column=col_idx, value=value)

# 数式
sheet.cell(row=4, column=2, value='=B2-B3')  # 利益
sheet.cell(row=4, column=1, value='利益')

# 列幅調整
sheet.column_dimensions['A'].width = 15
sheet.column_dimensions['B'].width = 15
sheet.column_dimensions['C'].width = 20

wb.save('output.xlsx')
```

---

## 既存ファイルの編集

```python
from openpyxl import load_workbook

wb = load_workbook('existing.xlsx')
sheet = wb.active  # or wb['シート名']

# セル編集
sheet['A1'] = '新しい値'

# 行挿入
sheet.insert_rows(2)

# 新しいシート追加
new_sheet = wb.create_sheet('集計')
new_sheet['A1'] = '=Sheet1!B2'  # クロスシート参照

wb.save('modified.xlsx')
```

**注意**: `data_only=True` で開くと数式が消える（計算結果のみ取得）。編集時は使わない。

---

## データ分析（pandas）

```python
import pandas as pd

# 読み込み
df = pd.read_excel('file.xlsx')
# 全シート: pd.read_excel('file.xlsx', sheet_name=None)
# 型指定: pd.read_excel('file.xlsx', dtype={'id': str})

# 基本統計
print(df.describe())
print(df.info())

# 集計
summary = df.groupby('カテゴリ')['売上'].agg(['sum', 'mean', 'count'])

# 出力
summary.to_excel('summary.xlsx')
```

---

## 財務モデルの色分けルール

ユーザーが財務モデルを作成する場合、以下の色分けを適用する：

| 色 | RGB | 用途 |
|----|-----|------|
| 青テキスト | (0,0,255) | ハードコード入力値、シナリオ変更用 |
| 黒テキスト | (0,0,0) | 全ての数式・計算 |
| 緑テキスト | (0,128,0) | 他シートからの参照 |
| 赤テキスト | (255,0,0) | 外部ファイルへのリンク |
| 黄背景 | (255,255,0) | 要注意の前提条件セル |

## 数値フォーマット

| 種別 | フォーマット |
|------|------------|
| 通貨 | `#,##0` （ヘッダーに単位明記: "売上(百万円)"） |
| ゼロ | `-` と表示: `#,##0;(#,##0);"-"` |
| パーセント | `0.0%` |
| 年度 | テキスト形式（"2026" を "2,026" にしない） |
| 負数 | 括弧表示 `(123)` |

---

## 数式エラー対策チェックリスト

- [ ] セル参照が正しいか（off-by-one注意）
- [ ] ゼロ除算がないか（`=IF(B2=0,0,A2/B2)`）
- [ ] クロスシート参照の形式（`Sheet1!A1`）
- [ ] 範囲指定が正しいか（行追加で範囲が足りなくならないか）
- [ ] 循環参照がないか

---

## 数式再計算

openpyxlで作成した数式は文字列として保存される（計算値は含まれない）。
LibreOfficeがある場合は再計算を実行：

```bash
/Applications/LibreOffice.app/Contents/MacOS/soffice --headless --calc --convert-to xlsx output.xlsx
```

---

## CSV/TSV からの変換

```python
import pandas as pd

# CSV読み込み（エンコーディング自動検出）
for enc in ['utf-8', 'shift_jis', 'cp932', 'euc-jp']:
    try:
        df = pd.read_csv('data.csv', encoding=enc)
        break
    except UnicodeDecodeError:
        continue

# Excel出力
df.to_excel('output.xlsx', index=False)
```

---

## 保存先

生成したファイルはユーザーに確認の上、適切な場所に保存する。
分析結果は `/分析メモ` コマンドでナレッジ化を提案する。
