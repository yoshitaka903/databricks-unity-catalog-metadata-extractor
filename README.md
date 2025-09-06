# Databricks Unity Catalog メタデータ抽出ツール

DatabricksのUnity Catalogから既存テーブルのメタデータを抽出・出力します。
テーブル一覧、カラム一覧、テーブルごとのサイズなど管理目的で利用できると思います。

## 主な機能

### 自動取得可能な情報
- **テーブル基本情報**: カタログ・スキーマ・テーブル名、タイプ、**所有者**、作成日時
- **物理ストレージ**: ストレージ形式、サイズ、ファイル数、ロケーション
- **パーティション**: パーティション戦略、パーティション列
- **クラスタリング**: Liquid Clustering、Z-Order最適化の検出
- **Delta機能**: CDF、自動最適化、保持設定、テーブル機能
- **カラム詳細**: データ型、制約、コメント、順序
- **PK/FK制約**: PK,FKの有無(あくまで参考情報)

## 使用方法

### 1. 前提条件
- 対象カタログへのUSAGE権限
- information_schemaへの読み取り権限

### 2. 実行手順

1. **ノートブックをDatabricksにインポート**
2. **セクション1-2**: 基本設定・対象スキーマの設定
3. **セクション3-4**: テーブル・カラム・制約情報を統合取得
4. **セクション5**: DESCRIBE DETAIL実行
5. **セクション6-7**: メタデータ構築・DataFrame化
6. **セクション8**: Delta テーブルとして保存

## 出力データ

### TABLE_DDL_INFO
| フィールド | 型 | 説明 |
|-----------|-----|------|
| catalog_name | string | カタログ名 |
| schema_name | string | スキーマ名 |
| table_name | string | テーブル名 |
| table_type | string | テーブルタイプ（MANAGED/EXTERNAL/VIEW） |
| table_owner | string | テーブル所有者 |
| storage_format | string | ストレージ形式（DELTA等、VIEWはnull） |
| partition_strategy | string | パーティション戦略（NONE/BY_COLUMNS） |
| clustering_strategy | string | クラスタリング戦略（NONE/LIQUID/ZORDER） |
| size_in_bytes | long | 物理サイズ（バイト） |
| size_pretty | string | 可読サイズ（例：1.2 GB） |
| table_features | array<string> | Delta機能リスト |
| auto_optimize_write | boolean | 自動最適化（書き込み） |
| cdf_enabled | boolean | Change Data Feed有効化 |

### COLUMN_DDL_INFO（統合版）
| フィールド | 型 | 説明 |
|-----------|-----|------|
| catalog_name | string | カタログ名 |
| schema_name | string | スキーマ名 |
| table_name | string | テーブル名 |
| column_name | string | カラム名 |
| data_type | string | データ型 |
| is_primary_key | boolean | Primary Keyフラグ |
| foreign_key_reference | string | FK参照有無（"FK参照あり" or null） |
| is_partition_column | boolean | パーティション列フラグ |
| is_clustering_column | boolean | クラスタリング列フラグ |
| is_nullable | boolean | NULL許可 |
| ordinal_position | int | カラム順序 |
