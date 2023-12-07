---
title: "BigQuery: ネストしたデータを平坦化する（UNNEST）"
emoji: "🧮"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["bigquery", "sql"]
published: false
---

## はじめに

## ネストしたデータを平坦化する (`ARRAY<STRUCT<X, Y>>` 型のケース)

```sql
WITH
  dummy_data AS (
    SELECT DATE("2023-04-01") AS date,
      [STRUCT("Rudisha" AS name, 1 AS rank),
      STRUCT("Makhloufi" AS name, 3 AS rank),
      STRUCT("Murphy" AS name, 2 AS rank),
      STRUCT("Bosse" AS name, 4 AS rank),
      STRUCT("Rotich" AS name, 6 AS rank),
      STRUCT("Lewandowski" AS name, 5 AS rank),
      STRUCT("Kipketer" AS name, 7 AS rank),
      STRUCT("Berian" AS name, 8 AS rank)]
        AS participants
    UNION ALL
    SELECT DATE("2023-04-02") AS date,
      [STRUCT("Rudisha" AS name, 8 AS rank),
      STRUCT("Makhloufi" AS name, 1 AS rank),
      STRUCT("Murphy" AS name, 3 AS rank),
      STRUCT("Bosse" AS name, 2 AS rank),
      STRUCT("Rotich" AS name, 4 AS rank),
      STRUCT("Lewandowski" AS name, 6 AS rank),
      STRUCT("Kipketer" AS name, 7 AS rank),
      STRUCT("Berian" AS name, 5 AS rank)]
        AS participants
  )
SELECT * FROM dummy_data

/*------------+-------------------+-------------------*
 | date       | participants.name | participants.rank |
 +------------+-------------------+-------------------|
 | 2023-04-01 | Rudisha           | 1                 |
 |            | Makhloufi         | 3                 |
 |            | Murphy            | 2                 |
 |            | Bosse             | 4                 |
 |            | Rotich            | 6                 |
 |            | Lewandowski       | 5                 |
 |            | Kipketer          | 7                 |
 |            | Berian            | 8                 |
 | 2023-04-02 | Rudisha           | 8                 |
 |            | Makhloufi         | 1                 |
 |            | Murphy            | 3                 |
 |            | Bosse             | 2                 |
 |            | Rotich            | 4                 |
 |            | Lewandowski       | 6                 |
 |            | Kipketer          | 7                 |
 |            | Berian            | 5                 |
 *------------+-------------------+-------------------*/
```

```sql
WITH
  dummy_data AS (
    SELECT DATE("2023-04-01") AS date,
      [STRUCT("Rudisha" AS name, 1 AS rank),
      STRUCT("Makhloufi" AS name, 3 AS rank),
      STRUCT("Murphy" AS name, 2 AS rank),
      STRUCT("Bosse" AS name, 4 AS rank),
      STRUCT("Rotich" AS name, 6 AS rank),
      STRUCT("Lewandowski" AS name, 5 AS rank),
      STRUCT("Kipketer" AS name, 7 AS rank),
      STRUCT("Berian" AS name, 8 AS rank)]
        AS participants
    UNION ALL
    SELECT DATE("2023-04-02") AS date,
      [STRUCT("Rudisha" AS name, 8 AS rank),
      STRUCT("Makhloufi" AS name, 1 AS rank),
      STRUCT("Murphy" AS name, 3 AS rank),
      STRUCT("Bosse" AS name, 2 AS rank),
      STRUCT("Rotich" AS name, 4 AS rank),
      STRUCT("Lewandowski" AS name, 6 AS rank),
      STRUCT("Kipketer" AS name, 7 AS rank),
      STRUCT("Berian" AS name, 5 AS rank)]
        AS participants
  )

SELECT
  dummy_data.date,
  participants.name,
  participants.rank,
FROM
  dummy_data,
  UNNEST(dummy_data.participants) AS participants

/*------------+-------------+------*
 | date       | name        | rank |
 +------------+-------------+------|
 | 2023-04-01 | Rudisha     | 1    |
 | 2023-04-01 | Makhloufi   | 3    |
 | 2023-04-01 | Murphy      | 2    |
 | 2023-04-01 | Bosse       | 4    |
 | 2023-04-01 | Rotich      | 6    |
 | 2023-04-01 | Lewandowski | 5    |
 | 2023-04-01 | Kipketer    | 7    |
 | 2023-04-01 | Berian      | 8    |
 | 2023-04-02 | Rudisha     | 8    |
 | 2023-04-02 | Makhloufi   | 1    |
 | 2023-04-02 | Murphy      | 3    |
 | 2023-04-02 | Bosse       | 2    |
 | 2023-04-02 | Rotich      | 4    |
 | 2023-04-02 | Lewandowski | 6    |
 | 2023-04-02 | Kipketer    | 7    |
 | 2023-04-02 | Berian      | 5    |
 *------------+-------------+------*/
```

## ネストしたデータを平坦化する (`ARRAY<X>` 型のケース)

```sql
WITH
  dummy_data AS (
    SELECT
      DATE("2023-04-01") AS date,
      ["HEADACHE", "IRRITATION"] AS symptoms
    UNION ALL
    SELECT
      DATE("2023-04-02") AS date,
      ["HEADACHE", "ASOMNIA", "DIZZINESS"] AS symptoms
  )

SELECT * FROM dummy_data

/*------------+------------*
 | date       | symptoms   |
 +------------+------------|
 | 2023-04-01 | HEADACHE   |
 |            | IRRITATION |
 | 2023-04-02 | HEADACHE   |
 |            | ASOMNIA    |
 |            | DIZZINESS  |
 *------------+------------*/
```
