---
name: PDF操作
description: "PDFファイルの結合・分割・変換・テキスト抽出・圧縮などの操作。Use when: 「PDF」「PDFを結合」「PDFを分割」「PDFから変換」"
argument-hint: "[操作内容 or ファイルパス]"
---

以下の手順でPDFファイルの操作を行ってください。

引数: $ARGUMENTS
（例: `/PDF操作 inbox/temp/資料.pdf のテキストを抽出して` `/PDF操作 3つのPDFを結合して` `/PDF操作 スキャンPDFをOCRして`）

---

## モード判定

`$ARGUMENTS` の内容から、以下のモードを自動判定する：

| モード | 判定条件 |
|--------|---------|
| テキスト抽出 | 「読み取り」「テキスト」「抽出」「中身」「内容」 |
| テーブル抽出 | 「表」「テーブル」「table」「データ」 |
| 結合 | 「結合」「マージ」「merge」「まとめ」 |
| 分割 | 「分割」「split」「ページごと」 |
| OCR | 「OCR」「スキャン」「画像PDF」「読めない」 |
| PDF作成 | 「作成」「生成」「create」「レポート」 |
| その他 | 回転、暗号化、透かし、画像抽出等 |

---

## 依存パッケージの確認

操作に必要なパッケージを事前に確認し、なければインストールする：

```python
# 基本（ほぼ全操作で使用）
# pip install pypdf pdfplumber

# テーブル抽出時
# pip install pandas openpyxl

# OCR時
# pip install pytesseract pdf2image
# brew install tesseract poppler

# PDF作成時
# pip install reportlab

# 日本語OCR時
# brew install tesseract-lang
```

---

## テキスト抽出

### 基本抽出（pdfplumber推奨 - レイアウト保持）

```python
import pdfplumber

with pdfplumber.open("document.pdf") as pdf:
    text = ""
    for i, page in enumerate(pdf.pages, 1):
        text += f"\n=== ページ {i} ===\n"
        text += page.extract_text() or "(テキストなし)"
        text += "\n"
```

### 軽量抽出（pypdf - 高速だがレイアウト簡易）

```python
from pypdf import PdfReader

reader = PdfReader("document.pdf")
for i, page in enumerate(reader.pages, 1):
    print(f"=== ページ {i} ===")
    print(page.extract_text())
```

---

## テーブル抽出

```python
import pdfplumber
import pandas as pd

with pdfplumber.open("document.pdf") as pdf:
    all_tables = []
    for page in pdf.pages:
        tables = page.extract_tables()
        for table in tables:
            if table:
                df = pd.DataFrame(table[1:], columns=table[0])
                all_tables.append(df)

    if all_tables:
        combined = pd.concat(all_tables, ignore_index=True)
        combined.to_excel("extracted_tables.xlsx", index=False)
        print(f"{len(all_tables)}個のテーブルを抽出しました")
```

---

## PDF結合

```python
from pypdf import PdfWriter, PdfReader

writer = PdfWriter()
for pdf_file in ["doc1.pdf", "doc2.pdf", "doc3.pdf"]:
    reader = PdfReader(pdf_file)
    for page in reader.pages:
        writer.add_page(page)

with open("merged.pdf", "wb") as output:
    writer.write(output)
```

---

## PDF分割

```python
from pypdf import PdfReader, PdfWriter

reader = PdfReader("input.pdf")
for i, page in enumerate(reader.pages, 1):
    writer = PdfWriter()
    writer.add_page(page)
    with open(f"page_{i}.pdf", "wb") as output:
        writer.write(output)
```

---

## OCR（スキャンPDF）

```python
import pytesseract
from pdf2image import convert_from_path

images = convert_from_path('scanned.pdf', dpi=300)

text = ""
for i, image in enumerate(images, 1):
    text += f"=== ページ {i} ===\n"
    # 日本語の場合は lang='jpn' を指定
    text += pytesseract.image_to_string(image, lang='jpn')
    text += "\n\n"
```

---

## PDF作成（reportlab）

### シンプルなPDF

```python
from reportlab.lib.pagesizes import A4
from reportlab.pdfgen import canvas

c = canvas.Canvas("output.pdf", pagesize=A4)
width, height = A4

c.drawString(100, height - 100, "タイトル")
c.drawString(100, height - 130, "本文テキスト")
c.line(100, height - 140, 500, height - 140)
c.save()
```

### 複数ページのレポート

```python
from reportlab.lib.pagesizes import A4
from reportlab.platypus import SimpleDocTemplate, Paragraph, Spacer, PageBreak
from reportlab.lib.styles import getSampleStyleSheet
from reportlab.pdfbase import pdfmetrics
from reportlab.pdfbase.ttfonts import TTFont

# 日本語フォント登録（必要に応じて）
# pdfmetrics.registerFont(TTFont('IPAGothic', '/path/to/ipag.ttf'))

doc = SimpleDocTemplate("report.pdf", pagesize=A4)
styles = getSampleStyleSheet()
story = []

story.append(Paragraph("レポートタイトル", styles['Title']))
story.append(Spacer(1, 12))
story.append(Paragraph("本文テキスト。" * 20, styles['Normal']))
story.append(PageBreak())
story.append(Paragraph("ページ2", styles['Heading1']))

doc.build(story)
```

### 注意事項
- Unicode下付き/上付き文字（数字など）は使わない。ReportLabの組み込みフォントで黒い四角になる
- 代わりに `<sub>` `<super>` タグを使う: `Paragraph("H<sub>2</sub>O", styles['Normal'])`

---

## その他の操作

### ページ回転
```python
from pypdf import PdfReader, PdfWriter

reader = PdfReader("input.pdf")
writer = PdfWriter()
page = reader.pages[0]
page.rotate(90)  # 時計回り90度
writer.add_page(page)
with open("rotated.pdf", "wb") as f:
    writer.write(f)
```

### パスワード保護
```python
writer.encrypt("userpassword", "ownerpassword")
```

### 画像抽出（コマンドライン）
```bash
pdfimages -j input.pdf output_prefix
```

---

## ナレッジ整理との連携

PDF からテキスト抽出した場合、ユーザーに以下を提案する：

```
抽出したテキストをナレッジとして保存しますか？
→ `/ナレッジ整理` で構造化して保存できます
```

保存先候補: `ナレッジ/{適切なカテゴリ}/` に `.md` で保存

---

## クイックリファレンス

| タスク | ベストツール | ポイント |
|--------|------------|---------|
| テキスト抽出 | pdfplumber | レイアウト保持 |
| テーブル抽出 | pdfplumber + pandas | DataFrameで操作 |
| 結合・分割 | pypdf | 軽量・高速 |
| OCR | pytesseract + pdf2image | 日本語は `lang='jpn'` |
| PDF作成 | reportlab | Canvas(簡易) / Platypus(複数ページ) |
| コマンドライン | qpdf / pdftotext | スクリプト不要な場合 |
