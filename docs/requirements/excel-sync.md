# Excel在庫同期

## 概要

管理画面からExcelをアップロードし、5分毎に自動でSupabase Storageから同期する仕組み。

---

## Excel列構成（後日確定）

想定される列：

- **商品名**（必須）
- **ケース数**（必須）
- **賞味期限**（任意）
- **ロット番号**（任意）
- **備考**（任意）

!!! warning "確認必要"
    Phase1実装前に、実際のExcel列構成の確定が必要です。

---

## 保存場所・同期方式

### Phase1: 手動アップロード

```mermaid
flowchart LR
    A[加工会社] -->|Excel更新| B[管理画面]
    B -->|アップロード| C[Supabase Storage]
    C -->|5分毎| D[Cron同期]
    D -->|UPSERT| E[inventory テーブル]
```

#### アップロード手順
1. 管理画面で「在庫Excelアップロード」ボタンをクリック
2. ファイル選択（.xlsx のみ）
3. アップロード実行
4. Supabase Storage の `inventory` bucket に保存
   - ファイル名: `inventory_YYYYMMDD_HHMMSS.xlsx`

#### 自動同期
- **実行頻度**: 5分毎
- **トリガー**: Supabase Cron
- **対象**: 最新ファイル

### Phase2: 自動アップロード（将来）

- Googleドライブ連携
- FTP自動取得
- API連携

---

## 同期処理フロー

```python
def sync_inventory_from_excel():
    # 1. 最新ファイルを取得
    latest_file = get_latest_file_from_storage('inventory')

    # 2. ファイル更新タイムスタンプを記録
    file_last_modified = get_file_timestamp(latest_file)

    # 3. Excelを読み込み（pandas）
    df = pd.read_excel(latest_file)

    # 4. 商品名→product_id照合
    for row in df.itertuples():
        product = match_product_by_name(row.商品名)

        if not product:
            log_error(f"商品名 '{row.商品名}' がマッチしません")
            continue

        # 5. inventoryへUPSERT
        upsert_inventory(
            product_id=product.id,
            qty_cases=row.ケース数,
            lot_no=row.ロット番号,
            expiry_date=row.賞味期限,
            status='provisional',
            source='excel_sync'
        )

    # 6. 同期ログ記録
    create_sync_log(
        file_path=latest_file,
        file_last_modified=file_last_modified,
        records_imported=len(df),
        status='success'
    )
```

---

## 商品名→product_id照合

### Phase1: 完全一致

```sql
SELECT id FROM products
WHERE name = '生チョコレート'
  AND is_active = true
LIMIT 1;
```

### 照合失敗時
- `inventory_sync_log.error_message` に記録
- 管理者LINEグループに通知

### Phase2: 部分一致・あいまい検索

- Levenshtein距離
- 類似度スコアリング
- 専用の照合エラーテーブル

---

## エラー通知

### 通知先
管理者LINEグループ

### 通知タイミング
- 同期失敗時（即時）
- 商品名照合失敗時（即時）

### 通知内容

```
【Excel同期エラー】
ファイル: inventory_20260215_143000.xlsx
エラー: 商品名 '生チョコ' がマッチしません

詳細は管理画面の同期ログをご確認ください。
```

---

## 履歴管理

### ファイル履歴
Supabase Storageに全ファイル保存（自動で履歴管理）

#### ファイル名規則
```
inventory/
  ├─ inventory_20260215_100000.xlsx
  ├─ inventory_20260215_143000.xlsx
  └─ inventory_20260215_180000.xlsx
```

### 同期ログ
`inventory_sync_log` テーブルで管理

#### 確認できる情報
- 同期実行時刻
- ファイル更新タイムスタンプ
- 取り込んだ行数
- 成功/失敗ステータス
- エラーメッセージ

---

## 在庫情報の鮮度表示

### タイムスタンプ表示

在庫照会時の返答：
```
【在庫照会結果】
商品: 生チョコレート
在庫: 10ケース
※この情報は2時間前のものです

納品日は別途ご連絡します。
```

### 計算ロジック

```python
def get_inventory_staleness():
    latest_sync = get_latest_sync_log()
    file_modified = latest_sync.file_last_modified
    now = datetime.now()

    hours_ago = (now - file_modified).total_seconds() / 3600
    return round(hours_ago)
```

### 警告閾値

`system_settings` で管理：

| key | value | 説明 |
|-----|-------|------|
| `excel_staleness_warning_hours` | `2` | 警告を表示する時間 |

---

## Phase2 改善計画

### 自動アップロード
- Googleドライブ連携
- Drive API で定期的にファイル取得
- 加工会社の運用を変えずに自動化

### エラー管理強化
- 専用エラーテーブル（`inventory_import_errors`）
- ダッシュボードでエラー一覧表示
- 自動リトライ機能

### 在庫 status 管理
- `confirmed` の本格運用
- 加工会社が確定操作
- provisional → confirmed の承認フロー
