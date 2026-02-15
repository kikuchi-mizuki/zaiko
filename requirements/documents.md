# 納品書生成

## 概要

出荷依頼時に納品書PDFを自動生成し、Supabase Storageに保存する機能。

---

## 既存フォーマット（後日確定）

### 確認が必要な項目

- [ ] 既存のExcel/PDFテンプレートファイルの入手
- [ ] 必須項目の確定
- [ ] 単価・金額の表示要否

### 想定される必須項目

**基本情報**:
- 納品日
- 納品書番号（管理番号）
- 発注元（高島屋○○店）
- 発注先（貴社名）

**商品情報**:
- 商品名
- 数量（ケース数 / 個数）
- 単価（要否は後日確定）
- 金額（要否は後日確定）
- 合計金額（要否は後日確定）

**その他**:
- ロット番号
- 賞味期限
- 備考欄
- 配送業者・追跡番号

---

## 生成タイミング

```
requests.status = 'shipment_requested'
```

になったタイミングで自動生成。

### フロー

```mermaid
flowchart LR
    A[本部承認] --> B[status: shipment_requested]
    B --> C[納品書PDF生成]
    C --> D[Supabase Storage保存]
    D --> E[documents テーブルに記録]
    E --> F[加工会社に通知]
```

---

## 保存先

### Supabase Storage

- **Bucket**: `documents`
- **パス**: `delivery_notes/{YYYY}/{MM}/{request_id}_v{version}.pdf`

### 例

```
documents/
  └─ delivery_notes/
      └─ 2026/
          └─ 02/
              ├─ abc123-def4-56gh-78ij-9klm01nop234_v1.pdf
              ├─ abc123-def4-56gh-78ij-9klm01nop234_v2.pdf
              └─ xyz789-uvw0-12ab-34cd-5efg67hij890_v1.pdf
```

---

## バージョン管理

### 再生成が必要なケース

- 数量修正
- 商品変更
- 配送先変更
- 再送依頼

### バージョン管理方法

```sql
-- 初回生成
INSERT INTO documents (
  request_id, doc_type, storage_path, version, is_latest
) VALUES (
  'abc123...', 'delivery_note', 'delivery_notes/2026/02/abc123..._v1.pdf', 1, true
);

-- 再生成時
UPDATE documents
SET is_latest = false
WHERE request_id = 'abc123...' AND is_latest = true;

INSERT INTO documents (
  request_id, doc_type, storage_path, version, is_latest
) VALUES (
  'abc123...', 'delivery_note', 'delivery_notes/2026/02/abc123..._v2.pdf', 2, true
);
```

### 管理画面での表示

```
納品書一覧
├─ v2（最新）[ダウンロード]
└─ v1       [ダウンロード]
```

---

## 実装方法（後日詳細確定）

### オプション1: reportlab

既存フォーマットがPDFの場合。

```python
from reportlab.pdfgen import canvas
from reportlab.lib.pagesizes import A4

def generate_delivery_note(request_id):
    # データ取得
    request = get_request_by_id(request_id)
    store = get_store_by_id(request.store_id)
    product = get_product_by_id(request.product_id)

    # PDF生成
    filename = f"/tmp/{request_id}_v1.pdf"
    c = canvas.Canvas(filename, pagesize=A4)

    # ヘッダー
    c.drawString(100, 800, f"納品書")
    c.drawString(100, 780, f"納品日: {datetime.now().strftime('%Y年%m月%d日')}")

    # 店舗情報
    c.drawString(100, 750, f"納品先: {store.name}")

    # 商品情報
    c.drawString(100, 700, f"商品名: {product.name}")
    c.drawString(100, 680, f"数量: {request.qty_requested}ケース")

    c.save()

    # Supabase Storageにアップロード
    upload_to_storage(filename, f"delivery_notes/2026/02/{request_id}_v1.pdf")
```

### オプション2: openpyxl → PDF

既存フォーマットがExcelの場合。

```python
from openpyxl import load_workbook
from openpyxl.utils import get_column_letter

def generate_delivery_note_from_template(request_id):
    # テンプレート読み込み
    wb = load_workbook('templates/delivery_note_template.xlsx')
    ws = wb.active

    # データ取得
    request = get_request_by_id(request_id)
    store = get_store_by_id(request.store_id)
    product = get_product_by_id(request.product_id)

    # データ埋め込み
    ws['B5'] = store.name  # 納品先
    ws['B10'] = product.name  # 商品名
    ws['C10'] = request.qty_requested  # 数量

    # 一時保存
    temp_xlsx = f"/tmp/{request_id}_v1.xlsx"
    wb.save(temp_xlsx)

    # PDF変換（libreoffice や別ツール使用）
    convert_xlsx_to_pdf(temp_xlsx, f"/tmp/{request_id}_v1.pdf")

    # Supabase Storageにアップロード
    upload_to_storage(f"/tmp/{request_id}_v1.pdf", f"delivery_notes/2026/02/{request_id}_v1.pdf")
```

---

## 単価・金額の扱い

### 単価表示が必要な場合

`products` テーブルに追加：

```sql
ALTER TABLE products
ADD COLUMN unit_price NUMERIC;
```

納品書に合計金額を記載：

```
商品名: 生チョコレート
数量: 10ケース
単価: ¥5,000
合計: ¥50,000
```

### 単価表示が不要な場合

数量のみ記載（請求は別途）。

---

## エラー処理

### PDF生成失敗時

```python
try:
    generate_delivery_note(request_id)
except Exception as e:
    # エラーログ記録
    update_request_error(request_id, str(e))

    # 管理者に通知
    send_notification_to_admin(
        f"納品書PDF生成失敗\nRequest ID: {request_id}\nエラー: {str(e)}"
    )
```

### リトライ

- 管理画面で「再生成」ボタン
- 手動でリトライ可能

---

## Phase2 改善計画

### テンプレートエンジン化
- Jinja2 などでテンプレート管理
- 帳票レイアウトの柔軟な変更

### 多言語対応
- 海外店舗向けに英語版納品書

### 電子署名
- PDF に電子署名を付与
- 改ざん防止
