---
title: "マーケット情報で遊ぶその2 - 株価情報で遊ぶ"
date: 2021-03-13T02:51:39+09:00
draft: false
---

ひとまず2021年分の日足チャートをBigQueryに突っ込んでみている。
データの収集に苦労していて一部のデータではあるけど、遊びがいがありそうなので。

### 2021年年始から30%以上伸びている銘柄
* 日足チャートの終値の月平均で比較

<!--more-->

```sql
SELECT
  companies.name,
  result.*,
  (diff / first_value) AS growth_rate
FROM (
  SELECT
    *
  FROM (
    SELECT
      base.code,
      base.average AS first_value,
      latest.average AS last_value,
      (latest.average - base.average) AS diff
    FROM (
      SELECT
        code,
        AVG(finish) AS average,
        EXTRACT(MONTH
        FROM
          DATE(date)) AS month
      FROM
        reizist-stocks.market.stocks
      WHERE
        EXTRACT(MONTH
        FROM
          DATE(date)) = 1
      GROUP BY
        code,
        EXTRACT(MONTH
        FROM
          DATE(date))) base
    INNER JOIN (
      SELECT
        *
      FROM (
        SELECT
          code,
          AVG(finish) AS average,
          EXTRACT(MONTH
          FROM
            DATE(date)) AS month
        FROM
          reizist-stocks.market.stocks
        WHERE
          EXTRACT(MONTH
          FROM
            DATE(date)) = 3
        GROUP BY
          code,
          EXTRACT(MONTH
          FROM
            DATE(date)))) latest
    ON
      base.code = latest.code)
  WHERE
    diff / first_value > 0.3) result
INNER JOIN
  reizist-stocks.market.companies
ON
  companies.code = result.code
```

```
name	code	first_value	last_value	diff	growth_rate
ＷｉｓｄｏｍＴｒｅｅ　ガソリン上場投資信託	1691	2074.7142857142862	2712.125	637.4107142857138	0.30722818976795396
ＮＥＸＴ　ＮＯＴＥＳ　ドバイ原油先物　ダブル・ブル　ＥＴＮ	2038	327.7894736842105	479.1111111111111	151.32163742690057	0.46164276029401263
国際石油開発帝石	1605	614.2105263157894	802.1111111111111	187.90058479532172	0.30592211749024106
三井松島ホールディングス	1518	768.1052631578948	1039.0	270.8947368421052	0.3526791832259832
寿スピリッツ	2222	5340.789473684211	7312.222222222223	1971.4327485380118	0.36912759026580166
クルーズ	2138	1267.7894736842102	2125.222222222222	857.432748538012	0.6763210819587443
ＣＡＩＣＡ	2315	17.263157894736842	58.333333333333336	41.07017543859649	2.3790650406504064
夢真ホールディングス	2362	696.0	978.0	282.0	0.4051724137931034
コシダカホールディングス	2157	419.3157894736843	555.0000000000001	135.68421052631584	0.3235847872473956
ツクイホールディングス	2398	557.5789473684212	922.3333333333333	364.7543859649121	0.6541753193631611
エスクリ	2196	308.15789473684214	482.8888888888889	174.73099415204678	0.5670177436189391
ルネサンス	2378	905.8947368421052	1180.888888888889	274.9941520467837	0.30356082319828553
ワールドホールディングス	2429	1910.2631578947369	2583.777777777778	673.514619883041	0.3525768777456338
```

前日比の値上がり率とかだと https://info.finance.yahoo.co.jp/ranking/?kd=1&tm=d&mk=1 で見れたりするけど、
前月比とか年単位の上昇幅みたいなデータはどこから参照するのがいいのかな。
日足チャートからこねくり回しているのは効率悪いことをしているんだろうなあという自覚はある。



補足: 日足チャートのBQスキーマ情報。

```
[
  {
    "mode": "REQUIRED",
    "name": "code",
    "type": "STRING"
  },
  {
    "mode": "REQUIRED",
    "name": "date",
    "type": "STRING"
  },
  {
    "mode": "REQUIRED",
    "name": "opening",
    "type": "INTEGER"
  },
  {
    "mode": "REQUIRED",
    "name": "high",
    "type": "INTEGER"
  },
  {
    "mode": "REQUIRED",
    "name": "low",
    "type": "INTEGER"
  },
  {
    "mode": "REQUIRED",
    "name": "finish",
    "type": "INTEGER"
  },
  {
    "mode": "REQUIRED",
    "name": "turnover",
    "type": "INTEGER"
  }
]
```

