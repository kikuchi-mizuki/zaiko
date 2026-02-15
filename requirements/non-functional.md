# 非機能要件

## 監査ログ

### 承認・差戻・取消の記録

`requests` テーブルで管理：

```sql
-- 誰が承認したか
approved_by_line_user_id  -- LINE経由
approved_by_admin_id      -- 管理画面経由
approved_at               -- 承認日時

-- 差戻し理由
rejected_reason           -- 差戻し理由
```

### 在庫同期履歴

`inventory_sync_log` テーブルで管理：

- 同期実行時刻
- ファイル更新タイムスタンプ
- 取り込んだ行数
- 成功/失敗ステータス
- エラーメッセージ

### 納品書生成履歴

`documents` テーブルで管理：

- 生成日時
- バージョン
- 生成者
- ストレージパス

---

## エラー通知

### 通知先

管理者LINEグループ

### 通知対象

1. **Excel同期失敗**
   - ファイル読み込みエラー
   - 商品名照合エラー
   - データ形式エラー

2. **PDF生成失敗**
   - テンプレート読み込みエラー
   - データ不足エラー
   - ストレージ保存エラー

3. **在庫引当失敗**
   - 在庫不足エラー
   - トランザクションエラー

### 記録方法

`requests.error_message` と `error_at` で記録：

```sql
UPDATE requests
SET error_message = '在庫引当失敗: 在庫不足',
    error_at = now()
WHERE id = 'abc123...';
```

---

## バックアップ

### Supabase標準バックアップ

- 自動バックアップ: 日次
- 保持期間: 7日間（Proプラン）
- ポイントインタイムリカバリ: 対応

### 定期export（要検討）

- CSV/JSONエクスポート
- S3などの外部ストレージに保存
- 頻度: 週次

---

## 同時アクセス

### 要件

- 繁忙期でもLINE問い合わせが詰まらない
- APIは軽量に設計

### 対策

1. **在庫照会の高速化**
   - インデックス最適化
   - キャッシュ活用（Redis検討）

2. **同一在庫への同時引当**
   - トランザクション制御
   - 排他ロック

```python
# 在庫引当時のトランザクション例
@transaction.atomic
def allocate_inventory(request_id, inventory_id, qty):
    # 排他ロック
    inventory = Inventory.objects.select_for_update().get(id=inventory_id)

    if inventory.qty_cases < qty:
        raise InsufficientInventoryError()

    # 在庫減算
    inventory.qty_cases -= qty
    inventory.save()

    # allocation 作成
    Allocation.objects.create(
        request_id=request_id,
        inventory_id=inventory_id,
        qty_allocated=qty,
        type='hold',
        status='active',
        expires_at=now() + timedelta(hours=24)
    )
```

---

## パフォーマンス

### 目標

| 操作 | 目標レスポンス時間 |
|------|-------------------|
| 在庫照会 | 3秒以内 |
| 追加発注（受付） | 5秒以内 |
| 承認処理 | 5秒以内 |
| 納品書PDF生成 | 10秒以内 |

### 最適化施策

1. **データベース**
   - 適切なインデックス作成
   - N+1クエリの回避
   - コネクションプーリング

2. **API**
   - 非同期処理（FastAPI の async/await）
   - ジョブキュー（納品書PDF生成など）

3. **ストレージ**
   - CDN導入（Phase2）
   - 署名付きURL のキャッシュ

---

## セキュリティ

### 認証

- **LINE**: LINE User ID ベース
- **管理画面**: Supabase Auth（Email + Password）

### 認可

- **RLS**: Supabase の Row Level Security で制御
- **店舗**: 自店舗のデータのみアクセス可
- **本部/管理者**: 全データアクセス可

### データ暗号化

- **通信**: HTTPS（TLS 1.2以上）
- **保存**: Supabase の標準暗号化（AES-256）

### 脆弱性対策

- **SQLインジェクション**: ORM/パラメータ化クエリ使用
- **XSS**: 管理画面では入力サニタイズ
- **CSRF**: トークン検証

---

## 可用性

### 目標

- **稼働率**: 99.9%（月間ダウンタイム: 約43分）
- **サービス時間**: 24時間365日

### Supabase のSLA

- 稼働率保証: 99.9%（Proプラン）
- 障害時の自動フェイルオーバー

### 監視

- **死活監視**: UptimeRobot など
- **エラー監視**: Sentry など（Phase2）
- **アラート**: 管理者LINEグループに通知

---

## スケーラビリティ

### Phase1（MVP）

- 同時接続: 100ユーザー程度
- リクエスト: 1000req/day 程度

### Phase2（拡張）

- 同時接続: 500ユーザー
- リクエスト: 10000req/day

### スケールアップ施策

1. **DB**: Supabase のプラン変更（Proプラン以上）
2. **Backend**: インスタンス数増加（水平スケール）
3. **ストレージ**: CDN導入

---

## 保守性

### ログ

- **アプリケーションログ**: 標準出力（Cloud Run等で自動収集）
- **監査ログ**: データベースに記録
- **エラーログ**: `requests.error_message` 等

### ドキュメント

- **要件定義書**: 本ドキュメント
- **API仕様書**: OpenAPI/Swagger（Phase2）
- **運用手順書**: Phase2で作成

### コード品質

- **型チェック**: mypy（Python）
- **リンター**: ruff（Python）
- **フォーマッター**: black（Python）
- **テスト**: pytest（Phase2で本格導入）
