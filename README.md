# Databricks Unity Catalog メタデータ抽出ツール

Unity Catalogから既存テーブルのメタデータを効率的に抽出し、TABLE_DDL_INFO と COLUMN_DDL_INFO 形式で整理するPythonツールです。

## 主な機能

### 自動取得可能な情報
- **テーブル基本情報**: カタログ・スキーマ・テーブル名、タイプ、作成日時
- **物理ストレージ**: ストレージ形式、サイズ、ファイル数、ロケーション
- **パーティション**: パーティション戦略、パーティション列
- **クラスタリング**: Liquid Clustering、Z-Order最適化の検出
- **Delta機能**: CDF、自動最適化、保持設定、テーブル機能
- **カラム詳細**: データ型、制約、コメント、順序
- **PK/FK制約**: Primary Key、Foreign Key参照関係 **NEW**

### パフォーマンス特化
- **並列処理**: ThreadPoolExecutor による高速DESCRIBE DETAIL実行
- **設定統合**: CONFIG辞書による一元管理
- **エラーハンドリング**: 個別テーブルエラーでも処理継続
- **進捗表示**: リアルタイム実行状況表示

## ファイル構成

```
databricks-unity-catalog-metadata-extractor/
├── README.md                                    # このファイル
├── unity_catalog_metadata_extractor.ipynb      # メインノートブック
└── .git/                                       # Git管理
```

## 使用方法

### 1. 前提条件
- Databricks Runtime 13.0+ (Unity Catalog対応)
- 対象カタログへのUSAGE権限
- information_schemaへの読み取り権限

### 2. 設定

```python
# CONFIG辞書で実行パラメータを設定
CONFIG = {
    "target_catalog": "samples",                    # 対象カタログ
    "include_schemas": ["tpch", "nyctaxi"],        # 対象スキーマリスト
    "output_catalog": "ops",                       # 出力先カタログ
    "exclude_patterns": ['^__', '^event_log_'],    # 除外パターン
    "table_types": ['MANAGED', 'EXTERNAL', 'VIEW'], # 対象テーブルタイプ
    "retention_days": 180,                         # 保持期間
    "max_parallel_workers": 4                      # 並列実行数
}
```

### 3. 実行手順

1. **ノートブックをDatabricksにインポート**
2. **セクション1-2**: 基本設定・対象スキーマ選択
3. **セクション3-4.5**: テーブル・カラム・制約情報取得
4. **セクション5-6**: DESCRIBE DETAIL並列実行
5. **セクション7-8**: データ構築・DataFrame化
6. **セクション9**: メタデータテーブル保存

## 出力データ

### TABLE_DDL_INFO
| フィールド | 型 | 説明 |
|-----------|-----|------|
| catalog_name | string | カタログ名 |
| table_name | string | テーブル名 |
| storage_format | string | ストレージ形式(DELTA等) |
| partition_strategy | string | パーティション戦略 |
| clustering_strategy | string | クラスタリング戦略(LIQUID/ZORDER) |
| size_in_bytes | long | 物理サイズ |
| table_features | array<string> | Delta機能リスト |

### COLUMN_DDL_INFO (PK/FK対応)
| フィールド | 型 | 説明 |
|-----------|-----|------|
| column_name | string | カラム名 |
| data_type | string | データ型 |
| is_primary_key | boolean | **Primary Keyフラグ** |
| foreign_key_reference | string | **FK参照先(catalog.schema.table.column)** |
| is_partition_column | boolean | パーティション列フラグ |
| is_clustering_column | boolean | クラスタリング列フラグ |

## 活用例

### 1. データカタログ構築
```sql
-- テーブルサイズTOP10
SELECT concat(catalog_name,'.',schema_name,'.',table_name) as full_name,
       size_pretty, clustering_strategy, partition_strategy
FROM ops.samples.samples_table_ddl_info
WHERE size_in_bytes IS NOT NULL
ORDER BY size_in_bytes DESC LIMIT 10;
```

### 2. PK/FK関係分析 (NEW)
```sql
-- FK参照関係の可視化
SELECT concat(catalog_name,'.',schema_name,'.',table_name,'.',column_name) as source_column,
       foreign_key_reference as target_reference
FROM ops.samples.samples_column_ddl_info
WHERE foreign_key_reference IS NOT NULL;
```

### 3. データ品質チェック
```sql
-- NOT NULL制約のないPKカラム検出
SELECT catalog_name, schema_name, table_name, column_name
FROM ops.samples.samples_column_ddl_info
WHERE is_primary_key = true AND is_nullable = true;
```

## 技術仕様

### アーキテクチャ
- **言語**: Python (PySpark)
- **並列処理**: ThreadPoolExecutor (最大4並列)
- **データソース**: information_schema、DESCRIBE DETAIL
- **出力形式**: Delta テーブル

### エラーハンドリング
- 個別テーブルエラーは`detail_error`フィールドに記録
- PK/FK制約が未サポートの環境でも動作継続
- 権限不足時の適切なスキップ処理

### パフォーマンス
- 1000テーブル: 約5-10分 (4並列)
- メモリ使用量: テーブル数×約1KB
- ネットワーク: DESCRIBE DETAIL実行回数に依存

## 制約・注意点

- Unity Catalog環境必須
- information_schema.table_constraintsの対応状況に依存
- 巨大テーブル(10,000+)では実行時間が長くなる可能性
- PK/FK制約情報は環境によって取得できない場合あり

## 更新履歴

- **v1.1** (2025-01-30): PK/FK制約情報取得機能追加
- **v1.0** (2025-01-15): 並列処理・設定統合・関数分割版リリース
- **v0.9** (2024-12-01): 初回リリース

## コントリビューション

改善提案やバグレポートはIssuesまでお願いします。

## ライセンス

MIT License