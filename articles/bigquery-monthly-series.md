---
title: "BigQuery: 歯抜けの日付データを保管する方法（GENERATE_DATE_ARRAY）"
emoji: "🧮"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["bigquery", "sql"]
published: true
publication_name: "cureapp"
---

## [基本] 単一のフィールド（例. 日付）をキーとして歯抜けとなったデータを保管する

以下のような歯抜けになった月次データがあったとします。
よく見ると、2019-07-01, 2019-08-01 などのデータが欠落しています。

```sql
WITH
  dummy_data AS (
    SELECT DATE('2019-04-01') as monthly, 51 as count UNION ALL
    SELECT DATE('2019-05-01'), 23, UNION ALL
    SELECT DATE('2019-06-01'), 45, UNION ALL
    SELECT DATE('2019-09-01'), 3, UNION ALL
    SELECT DATE('2019-12-01'), 70, UNION ALL
    SELECT DATE('2020-04-01'), 85, UNION ALL
    SELECT DATE('2021-04-01'), 77, UNION ALL
    SELECT DATE('2022-04-01'), 0, UNION ALL
    SELECT DATE('2023-04-01'), 1,
  )
SELECT * FROM dummy_data

/*------------+-------*
 | monthly    | count |
 +------------+-------|
 | 2019-04-01 | 51    |
 | 2019-05-01 | 23    |
 | 2019-06-01 | 45    |
 | 2019-09-01 | 3     |
 | 2019-12-01 | 70    |
 | 2020-04-01 | 85    |
 | 2021-04-01 | 77    |
 | 2022-04-01 | 0     |
 | 2023-04-01 | 1     |
 *------------+-------*/
```

こんなとき、月次の歯抜けデータをすべて以下のように count = 0 として保管したい場合ってありますよね。

```
/*------------+-------*
 | monthly    | count |
 +------------+-------|
 | 2019-04-01 | 51    |
 | 2019-05-01 | 23    |
 | 2019-06-01 | 45    |
 | 2019-07-01 | 0     | -> 保管したい
 | 2019-08-01 | 0     | -> 保管したい
 | 2019-09-01 | 3     |
 | 2019-10-01 | 0     | -> 保管したい
 | 2019-11-01 | 0     | -> 保管したい
 | 2019-12-01 | 70    |
 | ...        | ...   |
 *------------+-------*/
```

### 解決策

まず、`GENERATE_DATE_ARRAY` 関数を使って 2019/04/01 から月次の日付を生成します（`INTERVAL` の設定次第で、週次・日次でも可能です）

```sql
WITH
  monthly_series AS (
  SELECT
    monthly
  FROM
    UNNEST(GENERATE_DATE_ARRAY( DATE('2019-04-01'), CURRENT_DATE("Asia/Tokyo"), INTERVAL 1 MONTH)) AS monthly)

SELECT
  *
FROM
  monthly_series

/*------------*
 | monthly    |
 +------------|
 | 2019-04-01 |
 | 2019-05-01 |
 | 2019-06-01 |
 | 2019-07-01 |
 | 2019-08-01 |
 | 2019-09-01 |
 | 2019-10-01 |
 | 2019-11-01 |
 | 2019-12-01 |
 | ...        |
 *------------*/
```

上記で用意した月次リストのテーブル（`monthly_series`）とデータテーブル（`dummy_data`）を JOIN させます。
このとき、データテーブル側で歯抜けになっている場合、`count` が null になってしまうため、`IFNULL` 関数を使い、null だった場合 `0` に保管されるようにします。

```sql
WITH
  dummy_data AS (
    SELECT DATE('2019-04-01') as monthly, 51 as count UNION ALL
    SELECT DATE('2019-05-01'), 23, UNION ALL
    SELECT DATE('2019-06-01'), 45, UNION ALL
    SELECT DATE('2019-09-01'), 3, UNION ALL
    SELECT DATE('2019-12-01'), 70, UNION ALL
    SELECT DATE('2020-04-01'), 85, UNION ALL
    SELECT DATE('2021-04-01'), 77, UNION ALL
    SELECT DATE('2022-04-01'), 0, UNION ALL
    SELECT DATE('2023-04-01'), 1,
  ),
  monthly_series AS (
  SELECT
    monthly
  FROM
    UNNEST(GENERATE_DATE_ARRAY( DATE('2019-04-01'), CURRENT_DATE("Asia/Tokyo"), INTERVAL 1 MONTH)) AS monthly)

SELECT
  monthly_series.monthly,
  IFNULL(dummy_data.count, 0) AS count,
FROM
  monthly_series
LEFT OUTER JOIN
  dummy_data
ON
  monthly_series.monthly = dummy_data.monthly
```

### [応用] 複数のフィールドをキーとして歯抜けとなったデータを保管する

上記の例では、日付データのみをキーとして歯抜けとなったデータを保管しました。
これを応用して、複数のフィールドをキーとした場合でも保管が可能です。

例えば、以下の例のように「月次（`monthly`）」と「会社名（`company`）」をキーとして、歯抜けデータを保管したい場合を考えてみましょう。
（2019-06-01 の B 社のデータや、2019-07-01 の A 社のデータが歯抜けになっていますね）

```sql
WITH
  dummy_data AS (
    SELECT DATE('2019-04-01') as monthly, "A" as company, 51 as count UNION ALL
    SELECT DATE('2019-04-01'), "B", 23, UNION ALL
    SELECT DATE('2019-05-01'), "A", 45, UNION ALL
    SELECT DATE('2019-05-01'), "B", 3, UNION ALL
    SELECT DATE('2019-06-01'), "A", 70, UNION ALL
    SELECT DATE('2019-07-01'), "B", 85,
  )
SELECT * FROM dummy_data

/*------------+---------+-------*
 | monthly    | company | count |
 +------------+---------+-------|
 | 2019-04-01 | A       | 51    |
 | 2019-04-01 | B       | 23    |
 | 2019-05-01 | A       | 45    |
 | 2019-05-01 | B       | 3     |
 | 2019-06-01 | A       | 70    |
 | 2019-07-01 | B       | 85    |
 *------------+---------+-------*/
```

以下のようにデータを保管したいとします。

```
/*------------+---------+-------*
 | monthly    | company | count |
 +------------+---------+-------|
 | 2019-04-01 | A       | 51    |
 | 2019-04-01 | B       | 23    |
 | 2019-05-01 | A       | 45    |
 | 2019-05-01 | B       | 3     |
 | 2019-06-01 | A       | 70    |
 | 2019-06-01 | B       | 0     | -> 保管したい
 | 2019-07-01 | A       | 0     | -> 保管したい
 | 2019-07-01 | B       | 85    |
 *------------+---------+-------*/
```

### 解決策

まず、月次リストのテーブルと会社リストのテーブルを用意し、それらを `CROSS JOIN` させます。

```sql
WITH
  monthly_series AS (
    SELECT
      monthly
    FROM
      UNNEST(GENERATE_DATE_ARRAY( DATE('2019-04-01'), CURRENT_DATE("Asia/Tokyo"), INTERVAL 1 MONTH)) AS monthly
  ),
  companys AS (
    SELECT
      company
    FROM
      UNNEST(["A", "B"]) AS company
  ),
  monthly_company_list AS (
    SELECT
      monthly_series.monthly,
      companys.company,
    FROM
      -- CROSS JOIN
      monthly_series,
      companys
  )

SELECT
  *
FROM
  monthly_company_list

/*------------+---------*
 | monthly    | company |
 +------------+---------|
 | 2019-04-01 | A       |
 | 2019-04-01 | B       |
 | 2019-05-01 | A       |
 | 2019-05-01 | B       |
 | 2019-06-01 | A       |
 | 2019-06-01 | B       |
 | 2019-07-01 | A       |
 | 2019-07-01 | B       |
 | ...        | ...     |
 *------------+---------*/
```

上記で用意した月次 × 会社のクロスリストのテーブル（`monthly_company_list`）とデータテーブル（`dummy_data`）を JOIN させます。

```sql
WITH
  dummy_data AS (
    SELECT DATE('2019-04-01') as monthly, "A" as company, 51 as count UNION ALL
    SELECT DATE('2019-04-01'), "B", 23, UNION ALL
    SELECT DATE('2019-05-01'), "A", 45, UNION ALL
    SELECT DATE('2019-05-01'), "B", 3, UNION ALL
    SELECT DATE('2019-06-01'), "A", 70, UNION ALL
    SELECT DATE('2019-07-01'), "B", 85,
  ),
  monthly_series AS (
    SELECT
      monthly
    FROM
      UNNEST(GENERATE_DATE_ARRAY( DATE('2019-04-01'), CURRENT_DATE("Asia/Tokyo"), INTERVAL 1 MONTH)) AS monthly
  ),
  companys AS (
    SELECT
      company
    FROM
      UNNEST(["A", "B"]) AS company
  ),
  monthly_company_list AS (
    SELECT
      monthly_series.monthly,
      companys.company,
    FROM
      -- CROSS JOIN
      monthly_series,
      companys
  )

SELECT
  monthly_company_list.*,
  IFNULL(dummy_data.count, 0) AS count,
FROM
  monthly_company_list
LEFT OUTER JOIN
  dummy_data
ON
  monthly_company_list.monthly = dummy_data.monthly
  AND monthly_company_list.company = dummy_data.company

/*------------+---------+-------*
 | monthly    | company | count |
 +------------+---------+-------|
 | 2019-04-01 | A       | 51    |
 | 2019-04-01 | B       | 23    |
 | 2019-05-01 | A       | 45    |
 | 2019-05-01 | B       | 3     |
 | 2019-06-01 | A       | 70    |
 | 2019-06-01 | B       | 0     |
 | 2019-07-01 | A       | 0     |
 | 2019-07-01 | B       | 85    |
 | ...        | ...     | ...   |
 *------------+---------+-------*/
```
