---
description: 日本語CSVはUTF-8 BOM付きで保存（Excel文字化け対策）
globs:
---

# CSV出力のエンコーディング

日本語を含むCSVファイルを書き出すときは、**UTF-8 BOM付き** で保存する。

## 理由

- ExcelはBOMなしUTF-8を自動判定できず、Shift-JISとして解釈して文字化けする
- BOM付与で Mac/Windows のExcel・LibreOffice・Numbers すべて正しく開ける
- Shift-JIS変換は商品名に含まれる特殊文字（特殊括弧・髭文字など）で破損するため不可

## やり方

### Bashの場合

```bash
DST="output.csv"
TMP="${DST}.tmp"
printf '\xEF\xBB\xBF' > "$TMP"   # BOMを書く
cat "$DST" >> "$TMP"              # 本体を続ける
mv "$TMP" "$DST"
```

### Pythonの場合

```python
with open("output.csv", "w", encoding="utf-8-sig", newline="") as f:
    # utf-8-sig が自動でBOMを付ける
    writer = csv.writer(f)
    writer.writerow(["列名1", "列名2"])
```

### jq + bash で一気にやる場合

```bash
{ printf '\xEF\xBB\xBF'; jq -r '...' input.json; } > output.csv
```

## 確認方法

```bash
head -c 3 output.csv | xxd
# → 00000000: efbb bf       ...     これが出ればOK
```

## 例外

- 機械的に他システムへ流す中継ファイル（API → DBロードなど）はBOM無しを要求されることが多いので確認する
