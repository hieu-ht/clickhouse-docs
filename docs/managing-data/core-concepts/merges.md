---
slug: /merges
title: 'Part merges'
description: 'What are part merges in ClickHouse'
keywords: ['merges']
---

import merges_01 from '@site/static/images/managing-data/core-concepts/merges_01.png';
import merges_02 from '@site/static/images/managing-data/core-concepts/merges_02.png';
import merges_03 from '@site/static/images/managing-data/core-concepts/merges_03.png';
import merges_04 from '@site/static/images/managing-data/core-concepts/merges_04.png';
import merges_05 from '@site/static/images/managing-data/core-concepts/merges_05.png';
import merges_06 from '@site/static/images/managing-data/core-concepts/merges_06.png';
import merges_07 from '@site/static/images/managing-data/core-concepts/merges_07.png';
import merges_dashboard from '@site/static/images/managing-data/core-concepts/merges-dashboard.gif';
import Image from '@theme/IdealImage';

## What are part merges in ClickHouse? {#what-are-part-merges-in-clickhouse}

<br/>

ClickHouse [is fast](/concepts/why-clickhouse-is-so-fast) not just for queries but also for inserts, thanks to its [storage layer](https://www.vldb.org/pvldb/vol17/p3731-schulze.pdf), which operates similarly to [LSM trees](https://en.wikipedia.org/wiki/Log-structured_merge-tree):

① Inserts (into tables from the [MergeTree engine](/engines/table-engines/mergetree-family) family) create sorted, immutable [data parts](/parts).

② All data processing is offloaded to **background part merges**.

This makes data writes lightweight and [highly efficient](/concepts/why-clickhouse-is-so-fast#storage-layer-concurrent-inserts-are-isolated-from-each-other).

To control the number of parts per table and implement ② above, ClickHouse continuously merges ([per partition](/partitions#per-partition-merges)) smaller parts into larger ones in the background until they reach a compressed size of approximately [~150 GB](/operations/settings/merge-tree-settings#max_bytes_to_merge_at_max_space_in_pool).

The following diagram sketches this background merge process:

<Image img={merges_01} size="lg" alt='PART MERGES'/>

<br/>

The `merge level` of a part is incremented by one with each additional merge. A level of `0` means the part is new and has not been merged yet. Parts that were merged into larger parts are marked as [inactive](/operations/system-tables/parts) and finally deleted after a [configurable](/operations/settings/merge-tree-settings#old_parts_lifetime) time (8 minutes by default). Over time, this creates a **tree** of merged parts. Hence the name [merge tree](/engines/table-engines/mergetree-family) table.

## Monitoring merges {#monitoring-merges}

In the [what are table parts](/parts) example, we [showed](/parts#monitoring-table-parts) that ClickHouse tracks all table parts in the [parts](/operations/system-tables/parts) system table. We used the following query to retrieve the merge level and the number of stored rows per active part of the example table:
```sql
SELECT
    name,
    level,
    rows
FROM system.parts
WHERE (database = 'uk') AND (`table` = 'uk_price_paid_simple') AND active
ORDER BY name ASC;
```

The [previously documented](/parts#monitoring-table-parts) query result shows that the example table had four active parts, each created from a single merge of the initially inserted parts:
```response
   ┌─name────────┬─level─┬────rows─┐
1. │ all_0_5_1   │     1 │ 6368414 │
2. │ all_12_17_1 │     1 │ 6442494 │
3. │ all_18_23_1 │     1 │ 5977762 │
4. │ all_6_11_1  │     1 │ 6459763 │
   └─────────────┴───────┴─────────┘
```

[Running](https://sql.clickhouse.com/?query=U0VMRUNUCiAgICBuYW1lLAogICAgbGV2ZWwsCiAgICByb3dzCkZST00gc3lzdGVtLnBhcnRzCldIRVJFIChkYXRhYmFzZSA9ICd1aycpIEFORCAoYHRhYmxlYCA9ICd1a19wcmljZV9wYWlkX3NpbXBsZScpIEFORCBhY3RpdmUKT1JERVIgQlkgbmFtZSBBU0M7&run_query=true&tab=results) the query now shows that the four parts have since merged into a single final part (as long as there are no further inserts into the table):

```response
   ┌─name───────┬─level─┬─────rows─┐
1. │ all_0_23_2 │     2 │ 25248433 │
   └────────────┴───────┴──────────┘
```

In ClickHouse 24.10, a new [merges dashboard](https://presentations.clickhouse.com/2024-release-24.10/index.html#17) was added to the built-in [monitoring dashboards](https://clickhouse.com/blog/common-issues-you-can-solve-using-advanced-monitoring-dashboards). Available in both OSS and Cloud via the `/merges` HTTP handler, we can use it to visualize all part merges for our example table:

<Image img={merges_dashboard} size="lg" alt='PART MERGES'/>

<br/>

The recorded dashboard above captures the entire process, from the initial data inserts to the final merge into a single part:

① Number of active parts.

② Part merges, visually represented with boxes (size reflects part size).

③ [Write amplification](https://en.wikipedia.org/wiki/Write_amplification).

## Concurrent merges {#concurrent-merges}

A single ClickHouse server uses several background [merge threads](/operations/server-configuration-parameters/settings#background_pool_size) to execute concurrent part merges:

<Image img={merges_02} size="lg" alt='PART MERGES'/>

<br/>

Each merge thread executes a loop:

① Decide which parts to merge next, and load these parts into memory.

② Merge the parts in memory into a larger part.

③ Write the merged part to disk.

Go to ①

Note that increasing the number of CPU cores and the size of RAM allows to increase the background merge throughput.

## Memory optimized merges {#memory-optimized-merges}

ClickHouse does not necessarily load all parts to be merged into memory at once, as sketched in the [previous example](/merges#concurrent-merges). Based on several [factors](https://github.com/ClickHouse/ClickHouse/blob/bf37120c925ed846ae5cd72cd51e6340bebd2918/src/Storages/MergeTree/MergeTreeSettings.cpp#L210), and to reduce memory consumption (sacrificing merge speed), so-called [vertical merging](https://github.com/ClickHouse/ClickHouse/blob/bf37120c925ed846ae5cd72cd51e6340bebd2918/src/Storages/MergeTree/MergeTreeSettings.cpp#L209) loads and merges parts by chunks of blocks instead of in one go.

## Merge mechanics {#merge-mechanics}

The diagram below illustrates how a single background [merge thread](/merges#concurrent-merges) in ClickHouse merges parts (by default, without [vertical merging](/merges#memory-optimized-merges)):

<Image img={merges_03} size="lg" alt='PART MERGES'/>

<br/>

The part merging is performed in several steps:

**① Decompression & Loading**: The [compressed binary column files](/parts#what-are-table-parts-in-clickhouse) from the parts to be merged are decompressed and loaded into memory.

**② Merging**: The data is merged into larger column files.

**③ Indexing**: A new [sparse primary index](/guides/best-practices/sparse-primary-indexes) is generated for the merged column files.

**④ Compression & Storage**: The new column files and index are [compressed](/sql-reference/statements/create/table#column_compression_codec) and saved in a new [directory](/parts#what-are-table-parts-in-clickhouse) representing the merged data part.

Additional [metadata in data parts](/parts), such as secondary data skipping indexes, column statistics, checksums, and min-max indexes, is also recreated based on the merged column files. We omitted these details for simplicity.

The mechanics of step ② depend on the specific [MergeTree engine](/engines/table-engines/mergetree-family) used, as different engines handle merging differently. For example, rows may be aggregated or replaced if outdated. As mentioned earlier, this approach **offloads all data processing to background merges**, enabling **super-fast inserts** by keeping write operations lightweight and efficient.

Next, we will briefly outline the merge mechanics of specific engines in the MergeTree family.

### Standard merges {#standard-merges}

The diagram below illustrates how parts in a standard [MergeTree](/engines/table-engines/mergetree-family/mergetree) table are merged:

<Image img={merges_04} size="lg" alt='PART MERGES'/>

<br/>

The DDL statement in the diagram above creates a `MergeTree` table with a sorting key `(town, street)`, [meaning](/parts#what-are-table-parts-in-clickhouse) data on disk is sorted by these columns, and a sparse primary index is generated accordingly.

The ① decompressed, pre-sorted table columns are ② merged while preserving the table's global sorting order defined by the table's sorting key, ③ a new sparse primary index is generated, and ④ the merged column files and index are compressed and stored as a new data part on disk.

### Replacing merges {#replacing-merges}

Part merges in a [ReplacingMergeTree](/engines/table-engines/mergetree-family/replacingmergetree) table work similarly to [standard merges](/merges#standard-merges), but only the most recent version of each row is retained, with older versions being discarded:

<Image img={merges_05} size="lg" alt='PART MERGES'/>

<br/>

The DDL statement in the diagram above creates a `ReplacingMergeTree` table with a sorting key `(town, street, id)`, meaning data on disk is sorted by these columns, with a sparse primary index generated accordingly.

The ② merging works similarly to a standard `MergeTree` table, combining decompressed, pre-sorted columns while preserving the global sorting order.

However, the `ReplacingMergeTree` removes duplicate rows with the same sorting key, keeping only the most recent row based on the creation timestamp of its containing part.

<br/>

### Summing merges {#summing-merges}

Numeric data is automatically summarized during merges of parts from a [SummingMergeTree](/engines/table-engines/mergetree-family/summingmergetree) table:

<Image img={merges_06} size="lg" alt='PART MERGES'/>

<br/>

The DDL statement in the diagram above defines a `SummingMergeTree` table with `town` as the sorting key, meaning that data on disk is sorted by this column and a sparse primary index is created accordingly.

In the ② merging step, ClickHouse replaces all rows with the same sorting key with a single row, summing the values of numeric columns.

### Aggregating merges {#aggregating-merges}

The `SummingMergeTree` table example from above is a specialized variant of the [AggregatingMergeTree](/engines/table-engines/mergetree-family/aggregatingmergetree) table, allowing [automatic incremental data transformation](https://www.youtube.com/watch?v=QDAJTKZT8y4) by applying any of [90+](/sql-reference/aggregate-functions/reference) aggregation functions during part merges:

<Image img={merges_07} size="lg" alt='PART MERGES'/>

<br/>

The DDL statement in the diagram above creates an `AggregatingMergeTree` table with `town` as the sorting key, ensuring data is ordered by this column on disk and a corresponding sparse primary index is generated.

During ② merging, ClickHouse replaces all rows with the same sorting key with a single row storing [partial aggregation states](https://clickhouse.com/blog/clickhouse_vs_elasticsearch_mechanics_of_count_aggregations#-multi-core-parallelization) (e.g. a `sum` and a `count` for `avg()`). These states ensure accurate results through incremental background merges.
