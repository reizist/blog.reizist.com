---
title: "マーケット情報で遊ぶその1 -BigQueryに上場企業リスト作った"
date: 2021-03-12T02:11:47+09:00
draft: false
tags: ["engineering"]
---

上場銘柄一覧は [東京証券取引所のサイト](https://www.jpx.co.jp/markets/statistics-equities/misc/01.html)からダウンロードできる。

<!--more-->

BQのschema情報はこう。

```json
[
  {
    "mode": "REQUIRED",
    "name": "code",
    "type": "STRING"
  },
  {
    "mode": "REQUIRED",
    "name": "name",
    "type": "STRING"
  },
  {
    "mode": "REQUIRED",
    "name": "market",
    "type": "STRING"
  },
  {
    "mode": "NULLABLE",
    "name": "topix_33_code",
    "type": "STRING"
  },
  {
    "mode": "REQUIRED",
    "name": "topix_33",
    "type": "STRING"
  },
  {
    "mode": "NULLABLE",
    "name": "topix_17_code",
    "type": "STRING"
  },
  {
    "mode": "REQUIRED",
    "name": "topix_17",
    "type": "STRING"
  },
  {
    "mode": "NULLABLE",
    "name": "size_code",
    "type": "STRING"
  },
  {
    "mode": "NULLABLE",
    "name": "size",
    "type": "STRING"
  }
]
```

* *東証株価指数33業種* ごとの企業数

```sql
SELECT
  base.*,
  pure.topix_33
FROM (
  SELECT
    topix_33_code,
    COUNT(*) AS count
  FROM
    `reizist-stocks.market.companies`
  GROUP BY
    topix_33_code) base
INNER JOIN (
  SELECT
    DISTINCT(topix_33),
    topix_33_code
  FROM
    reizist-stocks.market.companies) pure
ON
  base.topix_33_code = pure.topix_33_code
ORDER BY
  count DESC
```

```
topix_33_code	count	topix_33
5250	503	情報・通信業
9050	493	サービス業
6100	351	小売業
6050	317	卸売業
3650	245	電気機器
3600	228	機械
3200	213	化学
2050	165	建設業
8050	139	不動産業
3050	123	食料品
3800	111	その他製品
3700	90	輸送用機器
3550	89	金属製品
7050	82	銀行業
3250	71	医薬品
5050	62	陸運業
3400	56	ガラス・土石製品
3100	53	繊維製品
3750	51	精密機器
3450	43	鉄鋼
7100	41	証券、商品先物取引業
5200	38	倉庫・運輸関連業
3500	35	非鉄金属
7200	33	その他金融業
3150	24	パルプ・紙
4050	24	電気・ガス業
3350	19	ゴム製品
7150	14	保険業
5100	13	海運業
50	12	水産・農林業
3300	11	石油・石炭製品
1050	6	鉱業
5150	5	空運業
```

* マーケットごとの企業数

```sql
SELECT
  market,
  COUNT(*) AS count
FROM
  reizist-stocks.market.companies
GROUP BY
  market
ORDER BY
  count DESC
```

```
market	count
市場第一部（内国株）	2194
JASDAQ(スタンダード・内国株）	666
市場第二部（内国株）	471
マザーズ（内国株）	345
ETF・ETN	264
REIT・ベンチャーファンド・カントリーファンド・インフラファンド	68
PRO Market	43
JASDAQ(グロース・内国株）	37
出資証券	2
市場第一部（外国株）	1
JASDAQ(スタンダード・外国株）	1
マザーズ（外国株）	1
市場第二部（外国株）	1
```

* [TOPIXニューインデックスシリーズ](https://www.jpx.co.jp/markets/indices/line-up/files/fac_12_size.pdf) 毎の企業数

```sql
SELECT
  size,
  COUNT(*) AS count
FROM
  reizist-stocks.market.companies
GROUP BY
  size
ORDER BY
  count DESC
```

```
size	count
-	            1905
TOPIX Small 2	1195
TOPIX Small 1	496
TOPIX Mid400	399
TOPIX Large70	70
TOPIX Core30	29
```

まだ何もわからないけどとりあえず好き勝手上場銘柄情報をいじれるようにはなった。



