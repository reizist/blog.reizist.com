---
title: "HerokuのFree planにprod dumpを使おうとしてハマったのでメモ"
date: 2021-08-23T16:44:22+09:00
draft: false
tags: ["engineering"]
---

二行で:

1. HerokuでRailsを動かす際にMySQLを無料プランで採用する場合、JawsDBが良さそう
2. 本番環境などのデータ量が多い環境からdumpをinsertする場合には挙動に注意

<!--more-->

## 詳しく

HerokuでMySQLを使いたい場合、 ClaerDBやJawsDB addonが使用できる。

が、ClearDBの場合、無料プランでデフォルトの環境だとMySQL5.6しか動かず、5.7へのupgradeは今日 2021/08/23 時点でできない。

```
reizist ...xxx/web upgrade-heroku-mysql-to-5.7* $ heroku addons:create cleardb:ignite --version=5.7 -a xxx-dev
 ›   Warning: heroku update available from 7.54.1 to 7.57.0.
Creating cleardb:ignite on ⬢ xxx-dev... free
 ▸    Release command executing: config vars set by this add-on will not be available until the command succeeds. Use `heroku
 ▸    releases:output` to view the log.
Created cleardb-infinite-55602 as CLEARDB_GREEN_URL
Use heroku addons:docs cleardb to view documentation
Time: 0h:00m:07s


reizist ...xxx/web upgrade-heroku-mysql-to-5.7* $ mysql -h us-cdbr-east-04.cleardb.com -u xxx -p
Enter password:
Welcome to the MySQL monitor.  Commands end with ; or \g.
Your MySQL connection id is 148347444
Server version: 5.6.50-log MySQL Community Server (GPL)

Copyright (c) 2000, 2021, Oracle and/or its affiliates.

Oracle is a registered trademark of Oracle Corporation and/or its
affiliates. Other names may be trademarks of their respective
owners.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

mysql> select version();
+------------+
| version()  |
+------------+
| 5.6.50-log |
+------------+
1 row in set (0.50 sec)
```

様々な理由で基本的にはMySQL5.6を今更使うべきではない([各所で5.6サポートはすでに停止している](https://aws.amazon.com/jp/blogs/news/amazon-rds-for-mysql-5-6-retirement/), charsetがutf8で絵文字対応を行う場合など基本的なところで詰まることが多いが5.7以降はdefault charsetがutf8mb4になっているので気にせずに済むなど)ので、
現時点においてはClearDBよりもJawsDBを使ったほうが効率が良さそう。

prod環境の（から機密情報をマスクした）データを流す場合、たいていaddonのFree planのストレージ限界は(5MB)オーバーするが、その際のJawsDBの挙動をメモしておく。

create dbした直後は、全ての権限を持っている。

```
mysql> show grants;
+------------------------------------------------------------------------+
| Grants for q08eeps477cixi15@%                                          |
+------------------------------------------------------------------------+ | GRANT USAGE ON *.* TO `q08eeps477cixi15`@`%`                           |
| GRANT ALL PRIVILEGES ON `jtq8b5ckygsy9ti5`.* TO `q08eeps477cixi15`@`%` |
+------------------------------------------------------------------------+
2 rows in set (0.19 sec)
```

が、dumpをimportした直後には以下のように権限状態が変化する。

```
mysql> show grants;
+----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| Grants for q08eeps477cixi15@%                                                                                                                                                                                                                |
+----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| GRANT USAGE ON *.* TO `q08eeps477cixi15`@`%`                                                                                                                                                                                                 |
| GRANT SELECT, UPDATE, DELETE, CREATE, DROP, REFERENCES, INDEX, ALTER, CREATE TEMPORARY TABLES, LOCK TABLES, EXECUTE, CREATE VIEW, SHOW VIEW, CREATE ROUTINE, ALTER ROUTINE, EVENT, TRIGGER ON `jtq8b5ckygsy9ti5`.* TO `q08eeps477cixi15`@`%` |
+--------------------------------------------------------------------------------------------------------------------------------------------
```

INSERT権限がなくなっており、例えば少し古いdumpをimport後にrails db:migrateをしようとすると、schema_migrationsにversionをinsertできずに失敗する。


