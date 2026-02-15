---
layout: default
title: トップページ
---

# 追加発注・在庫照会システム 仕様書

高島屋15店舗向けの追加発注・在庫照会・出荷依頼を自動化するLINEベースのシステムです。

---

## 📋 目次

### クライアント向け資料
- **[📄 在庫照会システム仕様書](system.md)** - システム概要を1ページで分かりやすく解説

### 詳細仕様書
1. [概要](requirements/overview.md) - プロジェクトの目的と方針
2. [システム構成](requirements/system.md) - 技術スタック・アーキテクチャ
3. [ユーザーと権限](requirements/users.md) - 役割定義・権限マトリクス
4. [業務フロー](requirements/workflow.md) - To-Be業務フロー・シーケンス図
5. [機能要件](requirements/features.md) - 各機能の詳細仕様
6. [LINE UI設計](requirements/line-ui.md) - チャットボットの会話設計
7. [データベース設計](requirements/database.md) - テーブル定義・ER図・RLS
8. [Excel在庫同期](requirements/excel-sync.md) - 在庫取り込み仕様
9. [納品書生成](requirements/documents.md) - 帳票自動生成
10. [非機能要件](requirements/non-functional.md) - パフォーマンス・セキュリティ
11. [フェーズ](requirements/phases.md) - 開発計画・ロードマップ
12. [未確定事項](requirements/todo.md) - 確認が必要な項目チェックリスト

---

## プロジェクト情報

| 項目 | 内容 |
|------|------|
| バージョン | v1.0 |
| 更新日 | 2026年2月15日 |
| ステータス | Phase1実装前（要件確定版） |
| GitHub | [kikuchi-mizuki/zaiko](https://github.com/kikuchi-mizuki/zaiko) |

---

## システム概要

### 目的
繁忙期における高島屋15店舗からの突発的な追加発注対応を効率化

### 主要機能
- 店舗がLINEで即座に在庫確認・発注できる
- 本部がスマホでも承認できる（外出先でもOK）
- 在庫の確度を管理し、事故を防ぐ
- 納品書を自動生成し、手作業を削減

### Phase1方針
**精度より運用の回りやすさを優先（暫定運用OK）**

---

## 次のステップ

1. [在庫照会システム仕様書](system.md)を確認
2. [未確定事項](requirements/todo.md)のチェックリストを確認
3. 必要な情報を提供
4. Phase1開発開始
