---
title: "BigQueryでカラムの型を変更する"
date: 2021-03-05T20:37:58+09:00
draft: false
---

BigQueryのWeb Console上からは一度定義したテーブルのスキーマ情報は更新ができず、カラム追加のみ可能となっている。

が、カラムの型を変更するときのオペレーションをさっくりメモ。

Doc は https://cloud.google.com/bigquery/docs/manually-changing-schemas にある。

## 1. バックアップ

```
bq cp --project_id target_project_id -f target_dataset.target_table target_dataset.target_table_copy
```

## 2. 型変換

```
bq query --project_id target_project_id --destination_table target_dataset.target_table --replace --use_legacy_sql=false 'SELECT
  * EXCEPT(target_column),
  CAST(target_column AS TARGET_TYPE) AS target_column
FROM
  target_dataset.target_table where _PARTITIONDATE>"2015-01-01"'
```

## 3. バックアップ消去

```
bq rm --project_id target_project_id target_dataset.target_table_copy
```
