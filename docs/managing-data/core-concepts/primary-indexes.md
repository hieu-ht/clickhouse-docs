---
slug: /primary-indexes
title: 'Primary indexes'
description: 'How does the sparse primary index work in ClickHouse'
keywords: ['sparse primary index', 'primary index', 'index']
---

import visual01 from '@site/static/images/managing-data/core-concepts/primary-index-light_01.gif';
import visual02 from '@site/static/images/managing-data/core-concepts/primary-index-light_02.gif';
import visual03 from '@site/static/images/managing-data/core-concepts/primary-index-light_03.gif';

import Image from '@theme/IdealImage';

:::tip Looking for advanced indexing details?
This page introduces ClickHouse's sparse primary index, how it's built, how it works, and how it helps accelerate queries.

For advanced indexing strategies and deeper technical detail, see the [primary indexes deep dive](/guides/best-practices/sparse-primary-indexes).
:::

## How does the sparse primary index work in ClickHouse? {#how-does-the-sparse-primary-index-work-in-clickHouse}

<br/>

The sparse primary index in ClickHouse helps efficiently identify [granules](https://clickhouse.com/docs/guides/best-practices/sparse-primary-indexes#data-is-organized-into-granules-for-parallel-data-processing)—blocks of rows—that might contain data matching a query's condition on the table's primary key columns. In the next section, we explain how this index is constructed from the values in those columns.

### Sparse primary index creation {#sparse-primary-index-creation}

To illustrate how the sparse primary index is built, we use the [uk_price_paid_simple](https://clickhouse.com/docs/parts) table along with some animations.

As a [reminder](https://clickhouse.com/docs/parts), in our ① example table with the primary key (town, street), ② inserted data is ③ stored on disk, sorted by the primary key column values, and compressed, in separate files for each column:

<Image img={visual01} size="lg"/>

<br/><br/>

For processing, each column's data is ④ logically divided into granules—each covering 8,192 rows—which are the smallest units ClickHouse's data processing mechanics work with.

This granule structure is also what makes the primary index **sparse**: instead of indexing every row, ClickHouse stores ⑤ the primary key values from just one row per granule—specifically, the first row. This results in one index entry per granule:

<Image img={visual02} size="lg"/>

<br/><br/>

Thanks to its sparseness, the primary index is small enough to fit entirely in memory, enabling fast filtering for queries with predicates on primary key columns. In the next section, we show how it helps accelerate such queries.

### Primary index usage {#primary-index-usage}

We sketch how the sparse primary index is used for query acceleration with another animation:

<Image img={visual03} size="lg"/>

<br/><br/>

① The example query includes a predicate on both primary key columns: `town = 'LONDON' AND street = 'OXFORD STREET'`.

② To accelerate the query, ClickHouse loads the table's primary index into memory.

③ It then scans the index entries to identify which granules might contain rows matching the predicate—in other words, which granules can't be skipped.

④ These potentially relevant granules are then loaded and [processed](/optimize/query-parallelism) in memory, along with the corresponding granules from any other columns required for the query.

## Monitoring primary indexes {#monitoring-primary-indexes}

Each [data part](/parts) in the table has its own primary index. We can inspect the contents of these indexes using the [mergeTreeIndex](/sql-reference/table-functions/mergeTreeIndex) table function.

The following query lists the number of entries in the primary index for each data part of our example table:

```sql
SELECT
    part_name,
    max(mark_number) AS entries
FROM mergeTreeIndex('uk', 'uk_price_paid_simple')
GROUP BY part_name;
```

```txt
   ┌─part_name─┬─entries─┐
1. │ all_2_2_0 │     914 │
2. │ all_1_1_0 │    1343 │
3. │ all_0_0_0 │    1349 │
   └───────────┴─────────┘
```

This query shows the first 10 entries from the primary index of one of the current data parts. Note that these parts are continuously [merged](/merges) in the background into larger parts:

```sql 
SELECT 
    mark_number + 1 AS entry,
    town,
    street
FROM mergeTreeIndex('uk', 'uk_price_paid_simple')
WHERE part_name = (SELECT any(part_name) FROM mergeTreeIndex('uk', 'uk_price_paid_simple')) 
ORDER BY mark_number ASC
LIMIT 10;
```

```txt
    ┌─entry─┬─town───────────┬─street───────────┐
 1. │     1 │ ABBOTS LANGLEY │ ABBEY DRIVE      │
 2. │     2 │ ABERDARE       │ RICHARDS TERRACE │
 3. │     3 │ ABERGELE       │ PEN Y CAE        │
 4. │     4 │ ABINGDON       │ CHAMBRAI CLOSE   │
 5. │     5 │ ABINGDON       │ THORNLEY CLOSE   │
 6. │     6 │ ACCRINGTON     │ MAY HILL CLOSE   │
 7. │     7 │ ADDLESTONE     │ HARE HILL        │
 8. │     8 │ ALDEBURGH      │ LINDEN ROAD      │
 9. │     9 │ ALDERSHOT      │ HIGH STREET      │
10. │    10 │ ALFRETON       │ ALMA STREET      │
    └───────┴────────────────┴──────────────────┘
```

Lastly, we use the [EXPLAIN](/sql-reference/statements/explain) clause to see how the primary indexes of all data parts are used to skip granules that can't possibly contain rows matching the example query's predicates. These granules are excluded from loading and processing:
```sql
EXPLAIN indexes = 1
SELECT
    max(price)
FROM
    uk.uk_price_paid_simple
WHERE
    town = 'LONDON' AND street = 'OXFORD STREET';
```

```txt
    ┌─explain────────────────────────────────────────────────────────────────────────────────────────────────────┐
 1. │ Expression ((Project names + Projection))                                                                  │
 2. │   Aggregating                                                                                              │
 3. │     Expression (Before GROUP BY)                                                                           │
 4. │       Expression                                                                                           │
 5. │         ReadFromMergeTree (uk.uk_price_paid_simple)                                                        │
 6. │         Indexes:                                                                                           │
 7. │           PrimaryKey                                                                                       │
 8. │             Keys:                                                                                          │
 9. │               town                                                                                         │
10. │               street                                                                                       │
11. │             Condition: and((street in ['OXFORD STREET', 'OXFORD STREET']), (town in ['LONDON', 'LONDON'])) │
12. │             Parts: 3/3                                                                                     │
13. │             Granules: 3/3609                                                                               │
    └────────────────────────────────────────────────────────────────────────────────────────────────────────────┘
```

Note how row 13 of the EXPLAIN output above shows that only 3 out of 3,609 granules across all data parts were selected by the primary index analysis for processing. The remaining granules were skipped entirely.

We can also observe that most of the data was skipped by simply running the query:
```sql 
SELECT max(price)
FROM uk.uk_price_paid_simple
WHERE (town = 'LONDON') AND (street = 'OXFORD STREET');
```

```txt
   ┌─max(price)─┐
1. │  263100000 │ -- 263.10 million
   └────────────┘

1 row in set. Elapsed: 0.010 sec. Processed 24.58 thousand rows, 159.04 KB (2.53 million rows/s., 16.35 MB/s.)
Peak memory usage: 13.00 MiB.
```

As shown above, only around 25,000 rows were processed out of approximately 30 million rows in the example table:
```sql 
SELECT count() FROM uk.uk_price_paid_simple;
```

```txt
   ┌──count()─┐
1. │ 29556244 │ -- 29.56 million
   └──────────┘
```

##  Key takeaways {#key-takeaways}

* **Sparse primary indexes** help ClickHouse skip unnecessary data by identifying which granules might contain rows matching query conditions on primary key columns. 

* Each index stores only the primary key values from the **first row of every granule** (a granule has 8,192 rows by default), making it compact enough to fit in memory. 

* **Each data part** in a MergeTree table has its **own primary index**, which is used independently during query execution. 

* During queries, the index allows ClickHouse to **skip granules**, reducing I/O and memory usage while accelerating performance. 

* You can **inspect index contents** using the `mergeTreeIndex` table function and monitor index usage with the `EXPLAIN` clause.

## Where to find more information {#where-to-find-more-information}

For a deeper look at how sparse primary indexes work in ClickHouse, including how they differ from traditional database indexes and best practices for using them, check out our detailed indexing [deep dive](/guides/best-practices/sparse-primary-indexes).

If you're interested in how ClickHouse processes data selected by the primary index scan in a highly parallel way, see the query parallelism guide [here](/optimize/query-parallelism).
