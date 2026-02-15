# システム構成

## 技術スタック

### フロント
- **LINE Official Account**（Messaging API）

### バックエンド
- **Python**（FastAPI想定）

### データベース
- **Supabase**（PostgreSQL + RLS）

### 認証
- **LINE**: LINE User ID ベース（`line_users` テーブル）
- **管理画面**: Supabase Auth（Email + Password）

### ファイルストレージ
Supabase Storage を使用：

- **納品書PDF**: `documents` bucket
- **在庫Excel**: `inventory` bucket

### ジョブ実行
Supabase Cron（または外部cron）で以下を実行：

- **Excel同期**: 5分毎
- **仮引当期限チェック**: 1時間毎

### 在庫入力元
- **Phase1**: Excel（手動アップロード）
- **Phase2**: 自動アップロード（Googleドライブ連携など）

### 帳票生成
- **PDF自動生成**: reportlab or openpyxl想定

## アーキテクチャ図

```
┌─────────────┐
│  店舗/本部  │
│   (LINE)    │
└──────┬──────┘
       │
       ↓
┌─────────────────────┐
│   FastAPI Backend   │
│  (Python)           │
└──────┬──────────────┘
       │
       ↓
┌─────────────────────┐
│    Supabase         │
│  ・PostgreSQL       │
│  ・Storage          │
│  ・Auth             │
│  ・Cron             │
└─────────────────────┘
```

## デプロイメント

### Phase1（想定）
- **Backend**: Cloud Run / Railway / Render
- **DB**: Supabase（クラウド）
- **ストレージ**: Supabase Storage
- **Cron**: Supabase Cron

### 将来の拡張
- スケールアウト対応
- CDN導入（PDF配信高速化）
- マルチリージョン対応
