---
name: ドキュメント作成
description: "Word文書(.docx)を作成・編集。テンプレートからの生成やフォーマット変換にも対応。Use when: 「Word」「docx」「文書作成」「ドキュメント」「Word文書」"
argument-hint: "[テーマ or ファイルパス]"
---

以下の手順でWord文書（.docx）を作成・編集してください。

引数: $ARGUMENTS
（例: `/ドキュメント作成 提案書のドラフトを作って` `/ドキュメント作成 議事録をWordに整形して` `/ドキュメント作成 既存のdocxを編集して`）

---

## モード判定

| モード | 判定条件 |
|--------|---------|
| 新規作成 | 「作成」「作って」「ドラフト」「Word化」「docxにして」 |
| 既存編集 | `.docx` パスの指定、「編集」「修正」「追加」 |
| 読み取り | 「読み取り」「テキスト抽出」「中身を見て」 |

---

## 依存パッケージ

```bash
npm install -g docx   # 新規作成用（docx-js）
pip install pypdf      # 読み取り補助
# pandoc は brew install pandoc でインストール
```

---

# 新規作成モード

`docx-js`（Node.js）を使って生成する。

## 基本テンプレート

```javascript
const fs = require('fs');
const { Document, Packer, Paragraph, TextRun, Table, TableRow, TableCell,
        Header, Footer, AlignmentType, HeadingLevel, LevelFormat,
        BorderStyle, WidthType, ShadingType, PageNumber, PageBreak,
        ExternalHyperlink, ImageRun } = require('docx');

const doc = new Document({
  styles: {
    default: { document: { run: { font: "Arial", size: 24 } } }, // 12pt
    paragraphStyles: [
      { id: "Heading1", name: "Heading 1", basedOn: "Normal", next: "Normal",
        quickFormat: true,
        run: { size: 32, bold: true, font: "Arial" },
        paragraph: { spacing: { before: 240, after: 240 }, outlineLevel: 0 } },
      { id: "Heading2", name: "Heading 2", basedOn: "Normal", next: "Normal",
        quickFormat: true,
        run: { size: 28, bold: true, font: "Arial" },
        paragraph: { spacing: { before: 180, after: 180 }, outlineLevel: 1 } },
    ]
  },
  sections: [{
    properties: {
      page: {
        size: { width: 12240, height: 15840 }, // US Letter (DXA)
        margin: { top: 1440, right: 1440, bottom: 1440, left: 1440 } // 1インチ
      }
    },
    headers: {
      default: new Header({ children: [
        new Paragraph({ children: [new TextRun({ text: "ヘッダー", size: 18, color: "999999" })] })
      ]})
    },
    footers: {
      default: new Footer({ children: [
        new Paragraph({
          alignment: AlignmentType.CENTER,
          children: [new TextRun("Page "), new TextRun({ children: [PageNumber.CURRENT] })]
        })
      ]})
    },
    children: [
      // ここにコンテンツを配置
    ]
  }]
});

Packer.toBuffer(doc).then(buffer => {
  fs.writeFileSync("output.docx", buffer);
  console.log("output.docx を作成しました");
});
```

## ページサイズ参考

| 用紙 | Width (DXA) | Height (DXA) | 1インチマージン時のコンテンツ幅 |
|------|-------------|-------------|--------------------------|
| US Letter | 12,240 | 15,840 | 9,360 |
| A4 (デフォルト) | 11,906 | 16,838 | 9,026 |

**注意**: docx-jsのデフォルトはA4。明示的に設定すること。

## 箇条書き（重要：Unicode弾丸文字は使わない）

```javascript
// NG: new Paragraph({ children: [new TextRun("・ 項目")] })

// OK: numbering config を使う
numbering: {
  config: [
    { reference: "bullets",
      levels: [{ level: 0, format: LevelFormat.BULLET, text: "\u2022",
        alignment: AlignmentType.LEFT,
        style: { paragraph: { indent: { left: 720, hanging: 360 } } } }]
    },
  ]
},
// 使用時:
new Paragraph({
  numbering: { reference: "bullets", level: 0 },
  children: [new TextRun("箇条書き項目")]
})
```

## テーブル（重要：幅を二重に設定）

```javascript
const border = { style: BorderStyle.SINGLE, size: 1, color: "CCCCCC" };
const borders = { top: border, bottom: border, left: border, right: border };

new Table({
  width: { size: 9360, type: WidthType.DXA },
  columnWidths: [4680, 4680], // 合計 = テーブル幅
  rows: [
    new TableRow({
      children: [
        new TableCell({
          borders,
          width: { size: 4680, type: WidthType.DXA }, // 各セルにも設定
          shading: { fill: "D5E8F0", type: ShadingType.CLEAR }, // CLEARを使う（SOLIDはNG）
          margins: { top: 80, bottom: 80, left: 120, right: 120 },
          children: [new Paragraph({ children: [new TextRun("セル")] })]
        })
      ]
    })
  ]
})
```

**テーブルの鉄則:**
- `WidthType.DXA` を使う（`PERCENTAGE` はGoogleドキュメントで崩れる）
- `columnWidths` の合計 = テーブル幅
- 各セルの `width` = 対応する `columnWidth`
- `ShadingType.CLEAR` を使う（`SOLID` は背景が黒になる）

## 画像

```javascript
new Paragraph({
  children: [new ImageRun({
    type: "png", // 必須: png, jpg, gif, bmp, svg
    data: fs.readFileSync("image.png"),
    transformation: { width: 200, height: 150 },
    altText: { title: "タイトル", description: "説明", name: "名前" }
  })]
})
```

## その他の注意事項

- `\n` は使わない。段落を分けて `new Paragraph()` を使う
- `PageBreak` は `Paragraph` の中に入れる
- 画像の `type` パラメータは必須
- テーブルを罫線代わりに使わない。`Paragraph` の `border.bottom` を使う

---

# 既存編集モード

.docxはZIPアーカイブ。XMLを直接編集する。

## 手順

### 1. テキスト抽出（内容確認）

```bash
# pandocがある場合
pandoc document.docx -o output.md

# Pythonの場合
python3 -c "
from zipfile import ZipFile
import xml.etree.ElementTree as ET

with ZipFile('document.docx') as z:
    tree = ET.parse(z.open('word/document.xml'))
    ns = {'w': 'http://schemas.openxmlformats.org/wordprocessingml/2006/main'}
    for p in tree.findall('.//w:p', ns):
        texts = [t.text for t in p.findall('.//w:t', ns) if t.text]
        if texts:
            print(''.join(texts))
"
```

### 2. XML展開・編集・再パック

```bash
# 展開
mkdir unpacked && cd unpacked
unzip ../document.docx

# word/document.xml を編集（Edit ツールで直接編集）

# 再パック
cd unpacked
zip -r ../output.docx . -x ".*"
```

### 3. LibreOfficeでPDF変換（必要な場合）

```bash
/Applications/LibreOffice.app/Contents/MacOS/soffice --headless --convert-to pdf document.docx
```

---

# 読み取りモード

```bash
# テキスト抽出
pandoc document.docx -t plain -o output.txt

# マークダウン変換
pandoc document.docx -o output.md
```

---

## 保存先

生成したドキュメントはユーザーに確認の上、適切な場所に保存する。
デフォルト候補: `inbox/` または指定されたパス。

## ドキュメントタイプ別のテンプレート選択

| タイプ | 推奨構成 |
|--------|---------|
| 提案書 | 表紙 → 目次 → エグゼクティブサマリー → 本文 → 付録 |
| 議事録 | ヘッダー（日時・参加者） → アジェンダ → 議論内容 → アクション |
| レポート | 表紙 → 概要 → 分析 → 結論 → 参考文献 |
| レター | レターヘッド → 宛先 → 本文 → 署名 |
