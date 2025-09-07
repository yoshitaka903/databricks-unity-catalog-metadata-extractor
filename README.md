# Unity Catalog メタデータ抽出ツール

Unity Catalogに格納されているテーブル・カラムのメタデータを抽出・管理するためのDatabricksノートブックです。

## 概要

Unity Catalogから以下のメタデータを抽出し、Delta Lakeテーブルとして保存します：

- **テーブル基本情報**: テーブル名、所有者、作成日時、最終更新日時
- **テーブル詳細設定**: パーティション、クラスタリング、Delta Lake設定
- **カラム情報**: データ型、制約、コメントなど
- **その他の設定**: Auto Optimization、保持期間設定など

## 主要機能

### 1. メタデータ抽出
- information_schemaを活用した効率的な基本情報取得
- DESCRIBE DETAILによる詳細テーブル設定の抽出
- 制約情報（主キー・外部キー）の統合取得

### 2. 高度な分析機能
- **Liquid Clustering**対応: 最新のクラスタリング戦略を検出
- **Z-ORDER検出**: 履歴から最近のZ-ORDER実行を確認（オプション）
- **Delta Lake設定解析**: Auto Optimization、CDF、統計設定の抽出
- **パーティション情報**: パーティション設計の識別

### 3. 運用・保守
- **設定管理**: CONFIG辞書による柔軟な設定制御
- **データ保持**: 設定可能な保持期間とVACUUM設定

## 設定

### 基本設定（CONFIG）

```python
CONFIG = {
    "target_catalog": "samples",              # 対象カタログ名
    "include_schemas": ["tpch", "nyctaxi"],   # 対象スキーマリスト
    "output_catalog": "ops",                  # 出力先カタログ
    "output_schema": "samples",               # 出力先スキーマ
    "exclude_patterns": ['^__', '^event_log_'], # 除外テーブルパターン
    "table_types": ['MANAGED', 'EXTERNAL', 'VIEW'], # 対象テーブル種別
    "retention_days": 180,                    # データ保持期間（日）
    "max_parallel_workers": 4,                # 並列処理数
    "detect_zorder_from_history": False,      # Z-ORDER検出(必要な場合)
    "zorder_lookback": 20,                   # 履歴確認件数(Z-ORDER検出用)
}
```

## 使用方法

### 1. 初期設定
```python
# 対象カタログを指定
catalog = "your_catalog_name"

# 設定を調整
CONFIG["target_catalog"] = catalog
CONFIG["include_schemas"] = ["schema1", "schema2"]  # 必要に応じて絞り込み
```

### 2. 実行
ノートブックの各セルを順番に実行：

1. **初期設定・パラメータ**：ライブラリ読み込みと設定
2. **テーブル基本情報取得**：対象テーブル一覧の抽出
3. **カラム基本情報取得**：カラム・制約情報の取得
4. **テーブル詳細情報取得**：DESCRIBE DETAIL実行（並列処理）
5. **メタデータ構築**：統合メタデータの作成
6. **メタデータ保存**：Deltaテーブルとして保存

### 3. 出力テーブル
以下のテーブルが作成されます：
- `{output_catalog}.{output_schema}.{target_catalog}_table_ddl_info`
- `{output_catalog}.{output_schema}.{target_catalog}_column_ddl_info`

## 出力スキーマ

### テーブル情報（TABLE_DDL_INFO）
| カラム名 | 説明 | データ型 |
|---------|------|----------|
| catalog_name | カタログ名 | STRING |
| schema_name | スキーマ名 | STRING |
| table_name | テーブル名 | STRING |
| table_type | テーブル種別 | STRING |
| storage_format | ストレージ形式 | STRING |
| partition_strategy | パーティション戦略 | STRING |
| partition_columns | パーティションカラム | ARRAY&lt;STRING&gt; |
| clustering_strategy | クラスタリング戦略 | STRING |
| clustering_columns | クラスタリングカラム | ARRAY&lt;STRING&gt; |
| auto_optimize_write | 書き込み最適化 | BOOLEAN |
| auto_optimize_compact | 自動コンパクション | BOOLEAN |
| cdf_enabled | Change Data Feed | BOOLEAN |
| vacuum_retention_hours | VACUUM保持時間 | INTEGER |
| stats_column_limit | 統計列数制限 | INTEGER |
| time_travel_retention_days | Time Travel保持日数 | INTEGER |
| num_files | ファイル数 | LONG |
| size_in_bytes | サイズ（バイト） | LONG |
| table_features | テーブル機能 | ARRAY&lt;STRING&gt; |

### カラム情報（COLUMN_DDL_INFO）
| カラム名 | 説明 | データ型 |
|---------|------|----------|
| catalog_name | カタログ名 | STRING |
| schema_name | スキーマ名 | STRING |
| table_name | テーブル名 | STRING |
| column_name | カラム名 | STRING |
| ordinal_position | カラム順序 | INTEGER |
| data_type | データ型 | STRING |
| is_nullable | NULL許可 | BOOLEAN |
| is_partition_column | パーティションカラム | BOOLEAN |
| is_clustering_column | クラスタリングカラム | BOOLEAN |
| is_primary_key | 主キー | BOOLEAN |
| foreign_key_reference | 外部キー参照 | STRING |

**注：DatabricksにおけるPK、FKは情報提供を目的としており、実際に制約は機能しないので注意してください**

## 利用時の考慮事項

### 1. パフォーマンスや取得する情報
- **対象の絞り込み**: `include_schemas`で必要なスキーマのみ処理
- **並列度調整**: `max_parallel_workers`をクラスターサイズに合わせて設定(デフォルト4)
- **Z-ORDER検出**: 必要な場合のみ`detect_zorder_from_history=True`

### 2. セキュリティ・ガバナンス
- 出力先カタログ・スキーマへの適切なアクセス権限設定
- 機密情報を含むテーブルの除外パターン設定
- 定期実行時のアクセス制御確認

### 3. 運用・保守
- **定期実行**: スケジューラーによる自動実行の設定
- **監視**: エラー発生時のアラート設定
- **データ保持**: ビジネス要件に応じた`retention_days`調整

## 制限事項・注意点

### 1. 権限要件
- 対象カタログ・スキーマへのUSE権限
- 各テーブルへのSELECT権限
- 出力先への書き込み権限

### 2. 互換性
- Unity Catalog有効環境でのみ動作
- information_schemaが利用可能な環境が必要
- Databricks Runtime 10.4 LTS以上を推奨

## 参考資料

- [Unity Catalog公式ドキュメント](https://docs.databricks.com/en/data-governance/unity-catalog/index.html)
- [information_schema参照](https://docs.databricks.com/en/sql/language-manual/sql-ref-information-schema.html)
- [Delta Lake テーブルプロパティ](https://docs.databricks.com/en/delta/table-properties.html)
- [パフォーマンス最適化ベストプラクティス](https://docs.databricks.com/en/lakehouse-architecture/performance-efficiency/best-practices.html)
