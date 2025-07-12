+++
title = "實習筆記 - Oracle 資料庫"
date = "2025-02-26T00:00:00+08:00"
#dateFormat = "2006-01-02" # This value can be configured for per-post date formatting
tags = ["實習筆記", "SQL"]
draft = true
+++

> 實習筆記補坑計劃 ep.0

<!--more-->

## 注意事項

- 並非每次都一定要依照正規化設計表格才是最好的
- 觀察原始資料結構、資料間的關聯、需求說明
- SELECT 查詢時指定欄位，不使用 `SELECT *`，可提升效能
- 實務上如果要JOIN表格比較常使用`LEFT JOIN`，而非`INNER JOIN`，因為通常會想要保留主表的所有欄位

## 資料庫索引與索引失效

### 使用索引的目的

- 加速 select 查詢，不用掃描整個資料表
- 提升比對資料的效能，例如 ORDER BY

### 使用索引的缺點

- 需要額外的儲存空間來建立索引
- 在某些情況下索引可能會成效不彰(例如資料量不大、用於建立索引的欄位有許多空值等)或是索引失效

### 索引失效的情況

1. 將萬用字元（%）置於關鍵詞前方（但後方可以），如 `SELECT ... FROM ...WHERE ... LIKE '%t'` 或是 `LIKE '%t%'`
2. 查詢中使用 `NOT`、`!=`、`NOT LIKE`
3. 查詢中使用 `OR`
4. 使用函數語句包裝被索引的欄位，例如 `SELECT ... FROM ...WHERE DATE(col) = '...'`當中使用了 `DATE(欄位名稱)`
5. ...

- [Database — 八個 Index( 索引 ) 無法生效的 SQL 寫法](https://medium.com/johnliu-%E7%9A%84%E8%BB%9F%E9%AB%94%E5%B7%A5%E7%A8%8B%E6%80%9D%E7%B6%AD/database-%E5%85%AB%E5%80%8B-index-%E7%B4%A2%E5%BC%95-%E7%84%A1%E6%B3%95%E7%94%9F%E6%95%88%E7%9A%84-sql-%E5%AF%AB%E6%B3%95-cdc7d2e72f51)
- https://www.cnblogs.com/12lisu/p/15786013.html

## PARTITION BY