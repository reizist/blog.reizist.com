---
title: "Datalake S3 Export Rds"
date: 2020-06-09T10:50:22+09:00
draft: false
---

データレイクをどうするかを考える機会があったのでシュッとメモ :memo:

<!--more-->

現在データマートを作るための、生データの参照先として、日時でとったsnapshotからrestoreしたDBを構築している(レポーティングDBと呼称している TODO: レポーティングDBとよんでいる理由?)。
が、DBのrunning costも馬鹿にならないし、マイクロサービスの文脈でボンボン新たなDBが映えるたびに1:1でレポーティングDBをボンボン増やすのもなぁ、ということで、S3をデータレイクにするパターンを検証した。

検証といってもやったことは単純で、[今年1月に発表されたRDS snapshotをS3にエクスポートする機能](https://aws.amazon.com/jp/about-aws/whats-new/2020/01/announcing-amazon-relational-database-service-snapshot-export-to-s3/)を使い、snapshotをparquet化。
Glueクローラを用いてS3からスキーマ検出/Athena上でテーブル化された状態で[embulk-input-athena](https://github.com/shinji19/embulk-input-athena)を用いてBQに投げることができるのを確認した。

唯一ハマったのは、S3へのexport時デフォルトでKMSによって暗号化されているためS3を参照するロールにKMSのDecrypt権限 `"kms:Decrypt"` が必要になる点。
