---
slug: /intro
sidebar_label: 'What is ClickHouse?'
description: 'ClickHouse® is a column-oriented SQL database management system (DBMS) for online analytical processing (OLAP). It is available as both an open-source software and a cloud offering.'
title: 'What is ClickHouse?'
---

import column_example from '@site/static/images/column-oriented-example-query.png';
import row_orientated from '@site/static/images/row-oriented.gif';
import column_orientated from '@site/static/images/column-oriented.gif';
import Image from '@theme/IdealImage';

ClickHouse® is a high-performance, column-oriented SQL database management system (DBMS) for online analytical processing (OLAP). It is available as both an [open-source software](https://github.com/ClickHouse/ClickHouse) and a [cloud offering](https://clickhouse.com/cloud).

## What are analytics? {#what-are-analytics}

Analytics, also known as OLAP (Online Analytical Processing), refers to SQL queries with complex calculations (e.g., aggregations, string processing, arithmetic) over massive datasets.

Unlike transactional queries (or OLTP, Online Transaction Processing) that read and write just a few rows per query and, therefore, complete in milliseconds, analytics queries routinely process billions and trillions of rows.

In many use cases, [analytics queries must be "real-time"](https://clickhouse.com/engineering-resources/what-is-real-time-analytics), i.e., return a result in less than one second.

## Row-oriented vs. column-oriented storage {#row-oriented-vs-column-oriented-storage}

Such a level of performance can only be achieved with the right data "orientation".

Databases store data either [row-oriented or column-oriented](https://clickhouse.com/engineering-resources/what-is-columnar-database).

In a row-oriented database, consecutive table rows are sequentially stored one after the other. This layout allows to retrieve rows quickly as the column values of each row are stored together.

ClickHouse is a column-oriented databases. In such systems, tables are stored as a collection of columns, i.e. the values of each column are stored sequentially one after the other. This layout makes it harder to restore single rows (as there are now gaps between the row values) but column operations such as filters or aggregation becomes much faster than in a row-oriented database.

The difference is best explained with an example query running over 100 million rows of [real-world anonymized web analytics data](/getting-started/example-datasets/metrica):

```sql
SELECT MobilePhoneModel, COUNT() AS c
FROM metrica.hits
WHERE
      RegionID = 229
  AND EventDate >= '2013-07-01'
  AND EventDate <= '2013-07-31'
  AND MobilePhone != 0
  AND MobilePhoneModel not in ['', 'iPad']
GROUP BY MobilePhoneModel
ORDER BY c DESC
LIMIT 8;
```

You can [run this query on the ClickHouse SQL Playground](https://sql.clickhouse.com?query=U0VMRUNUIE1vYmlsZVBob25lTW9kZWwsIENPVU5UKCkgQVMgYyAKRlJPTSBtZXRyaWNhLmhpdHMgCldIRVJFIAogICAgICBSZWdpb25JRCA9IDIyOSAKICBBTkQgRXZlbnREYXRlID49ICcyMDEzLTA3LTAxJyAKICBBTkQgRXZlbnREYXRlIDw9ICcyMDEzLTA3LTMxJyAKICBBTkQgTW9iaWxlUGhvbmUgIT0gMCAKICBBTkQgTW9iaWxlUGhvbmVNb2RlbCBub3QgaW4gWycnLCAnaVBhZCddIApHUk9VUCBCWSBNb2JpbGVQaG9uZU1vZGVsCk9SREVSIEJZIGMgREVTQyAKTElNSVQgODs&chart=eyJ0eXBlIjoicGllIiwiY29uZmlnIjp7InhheGlzIjoiTW9iaWxlUGhvbmVNb2RlbCIsInlheGlzIjoiYyJ9fQ&run_query=true) that selects and filters [just a few out of over 100](https://sql.clickhouse.com/?query=U0VMRUNUIG5hbWUKRlJPTSBzeXN0ZW0uY29sdW1ucwpXSEVSRSBkYXRhYmFzZSA9ICdtZXRyaWNhJyBBTkQgdGFibGUgPSAnaGl0cyc7&tab=results&run_query=true) existing columns, returning the result within milliseconds:

<Image img={column_example} alt="Example query in a column-oriented database" size="lg"/>

As you can see in the stats section in the above diagram, the query processed 100 million rows in 92 milliseconds, a throughput of approximately over 1 billion rows per second or just under 7 GB of data transferred per second.

**Row-oriented DBMS**

In a row-oriented database, even though the query above only processes a few out of the existing columns, the system still needs to load the data from other existing columns from disk to memory. The reason for that is that data is stored on disk in chunks called [blocks](https://en.wikipedia.org/wiki/Block_(data_storage)) (usually fixed sizes, e.g., 4 KB or 8 KB). Blocks are the smallest units of data read from disk to memory. When an application or database requests data, the operating system's disk I/O subsystem reads the required blocks from the disk. Even if only part of a block is needed, the entire block is read into memory (this is due to disk and file system design):

<Image img={row_orientated} alt="Row-oriented database structure" size="lg"/>

**Column-oriented DBMS**

Because the values of each column are stored sequentially one after the other on disk, no unnecessary data is loaded when the query from above is run.
Because the block-wise storage and transfer from disk to memory is aligned with the data access pattern of analytical queries, only the columns required for a query are read from disk, avoiding unnecessary I/O for unused data. This is [much faster](https://benchmark.clickhouse.com/) compared to row-based storage, where entire rows (including irrelevant columns) are read:

<Image img={column_orientated} alt="Column-oriented database structure" size="lg"/>

## Data replication and integrity {#data-replication-and-integrity}

ClickHouse uses an asynchronous multi-master replication scheme to ensure that data is stored redundantly on multiple nodes. After being written to any available replica, all the remaining replicas retrieve their copy in the background. The system maintains identical data on different replicas. Recovery after most failures is performed automatically, or semi-automatically in complex cases.

## Role-Based Access Control {#role-based-access-control}

ClickHouse implements user account management using SQL queries and allows for role-based access control configuration similar to what can be found in ANSI SQL standard and popular relational database management systems.

## SQL support {#sql-support}

ClickHouse supports a [declarative query language based on SQL](/sql-reference) that is identical to the ANSI SQL standard in many cases. Supported query clauses include [GROUP BY](/sql-reference/statements/select/group-by), [ORDER BY](/sql-reference/statements/select/order-by), subqueries in [FROM](/sql-reference/statements/select/from), [JOIN](/sql-reference/statements/select/join) clause, [IN](/sql-reference/operators/in) operator, [window functions](/sql-reference/window-functions) and scalar subqueries.

## Approximate calculation {#approximate-calculation}

ClickHouse provides ways to trade accuracy for performance. For example, some of its aggregate functions calculate the distinct value count, the median, and quantiles approximately. Also, queries can be run on a sample of the data to compute an approximate result quickly. Finally, aggregations can be run with a limited number of keys instead of for all keys. Depending on how skewed the distribution of the keys is, this can provide a reasonably accurate result that uses far fewer resources than an exact calculation.

## Adaptive join algorithms {#adaptive-join-algorithms}

ClickHouse chooses the join algorithm adaptively, it starts with fast hash joins and falls back to merge joins if there's more than one large table.

## Superior query performance {#superior-query-performance}

ClickHouse is well known for having extremely fast query performance.
To learn why ClickHouse is so fast, see the [Why is ClickHouse fast?](/concepts/why-clickhouse-is-so-fast.md) guide.

<!--
## What is OLAP? {#what-is-olap}
OLAP scenarios require real-time responses on top of large datasets for complex analytical queries with the following characteristics:
- Datasets can be massive - billions or trillions of rows
- Data is organized in tables that contain many columns
- Only a few columns are selected to answer any particular query
- Results must be returned in milliseconds or seconds

## Column-oriented vs row-oriented databases {#column-oriented-vs-row-oriented-databases}
In a row-oriented DBMS, data is stored in rows, with all the values related to a row physically stored next to each other.

In a column-oriented DBMS, data is stored in columns, with values from the same columns stored together.

## Why column-oriented databases work better in the OLAP scenario {#why-column-oriented-databases-work-better-in-the-olap-scenario}

Column-oriented databases are better suited to OLAP scenarios: they are at least 100 times faster in processing most queries. The reasons are explained in detail below, but the fact is easier to demonstrate visually:

See the difference?

The rest of this article explains why column-oriented databases work well for these scenarios, and why ClickHouse in particular [outperforms](/concepts/why-clickhouse-is-so-fast/concepts/why-clickhouse-is-so-fast#storage-layer-concurrent-inserts-and-selects-are-isolated) others in this category.

## Why is ClickHouse so fast? {#why-is-clickhouse-so-fast}

ClickHouse uses all available system resources to their full potential to process each analytical query as fast as possible. This is made possible due to a unique combination of analytical capabilities and attention to the low-level details required to implement the fastest OLAP database.

Helpful articles to dive deeper into this topic include:
- [ClickHouse Performance](/concepts/why-clickhouse-is-so-fast)
- [Distinctive Features of ClickHouse](/about-us/distinctive-features.md)
- [FAQ: Why is ClickHouse so fast?](/knowledgebase/why-clickhouse-is-so-fast)

## Processing analytical queries in real time {#processing-analytical-queries-in-real-time}

In a row-oriented DBMS, data is stored in this order:

| Row | WatchID     | JavaEnable | Title              | GoodEvent | EventTime           |
|-----|-------------|------------|--------------------|-----------|---------------------|
| #0 | 89354350662 | 1          | Investor Relations | 1         | 2016-05-18 05:19:20 |
| #1 | 90329509958 | 0          | Contact us         | 1         | 2016-05-18 08:10:20 |
| #2 | 89953706054 | 1          | Mission            | 1         | 2016-05-18 07:38:00 |
| #N | ...           | ...          | ...                  | ...         | ...                   |

In other words, all the values related to a row are physically stored next to each other.

Examples of a row-oriented DBMS are MySQL, Postgres, and MS SQL Server.

In a column-oriented DBMS, data is stored like this:

| Row:        | #0                 | #1                 | #2                 | #N |
|-------------|---------------------|---------------------|---------------------|-----|
| WatchID:    | 89354350662         | 90329509958         | 89953706054         | ...   |
| JavaEnable: | 1                   | 0                   | 1                   | ...   |
| Title:      | Investor Relations  | Contact us          | Mission             | ...   |
| GoodEvent:  | 1                   | 1                   | 1                   | ...   |
| EventTime:  | 2016-05-18 05:19:20 | 2016-05-18 08:10:20 | 2016-05-18 07:38:00 | ...   |

These examples only show the order that data is arranged in. The values from different columns are stored separately, and data from the same column is stored together.

Examples of a column-oriented DBMS: Vertica, Paraccel (Actian Matrix and Amazon Redshift), Sybase IQ, Exasol, Infobright, InfiniDB, MonetDB (VectorWise and Actian Vector), LucidDB, SAP HANA, Google Dremel, Google PowerDrill, Druid, and kdb+.

Different orders for storing data are better suited to different scenarios. The data access scenario refers to what queries are made, how often, and in what proportion; how much data is read for each type of query – rows, columns, and bytes; the relationship between reading and updating data; the working size of the data and how locally it is used; whether transactions are used, and how isolated they are; requirements for data replication and logical integrity; requirements for latency and throughput for each type of query, and so on.

The higher the load on the system, the more important it is to customize the system set up to match the requirements of the usage scenario, and the more fine grained this customization becomes. There is no system that is equally well-suited to significantly different scenarios. If a system is adaptable to a wide set of scenarios, under a high load, the system will handle all the scenarios equally poorly, or will work well for just one or few of possible scenarios.

### Key properties of the OLAP scenario {#key-properties-of-olap-scenario}

- Tables are "wide," meaning they contain a large number of columns.
- Datasets are large and queries require high throughput when processing a single query (up to billions of rows per second per server).
- Column values are fairly small: numbers and short strings (for example, 60 bytes per URL).
- Queries extract a large number of rows, but only a small subset of columns.
- For simple queries, latencies around 50ms are allowed.
- There is one large table per query; all tables are small, except for one.
- A query result is significantly smaller than the source data. In other words, data is filtered or aggregated, so the result fits in a single server's RAM.
- Queries are relatively rare (usually hundreds of queries per server or less per second).
- Inserts happen in fairly large batches (\> 1000 rows), not by single rows.
- Transactions are not necessary.

It is easy to see that the OLAP scenario is very different from other popular scenarios (such as OLTP or Key-Value access). So it does not make sense to try to use OLTP or a Key-Value DB for processing analytical queries if you want to get decent performance. For example, if you try to use MongoDB or Redis for analytics, you will get very poor performance compared to OLAP databases.

### Input/output {#inputoutput}

1.  For an analytical query, only a small number of table columns need to be read. In a column-oriented database, you can read just the data you need. For example, if you need 5 columns out of 100, you can expect a 20-fold reduction in I/O.
2.  Since data is read in packets, it is easier to compress. Data in columns is also easier to compress. This further reduces the I/O volume.
3.  Due to the reduced I/O, more data fits in the system cache.

For example, the query "count the number of records for each advertising platform" requires reading one "advertising platform ID" column, which takes up 1 byte uncompressed. If most of the traffic was not from advertising platforms, you can expect at least 10-fold compression of this column. When using a quick compression algorithm, data decompression is possible at a speed of at least several gigabytes of uncompressed data per second. In other words, this query can be processed at a speed of approximately several billion rows per second on a single server. This speed is actually achieved in practice.

### CPU {#cpu}

Since executing a query requires processing a large number of rows, it helps to dispatch all operations for entire vectors instead of for separate rows, or to implement the query engine so that there is almost no dispatching cost. If you do not do this, with any half-decent disk subsystem, the query interpreter inevitably stalls the CPU. It makes sense to both store data in columns and process it, when possible, by columns.

There are two ways to do this:

1.  A vector engine. All operations are written for vectors, instead of for separate values. This means you do not need to call operations very often, and dispatching costs are negligible. Operation code contains an optimized internal cycle.

2.  Code generation. The code generated for the query has all the indirect calls in it.

This is not done in row-oriented databases, because it does not make sense when running simple queries. However, there are exceptions. For example, MemSQL uses code generation to reduce latency when processing SQL queries. (For comparison, analytical DBMSs require optimization of throughput, not latency.)

Note that for CPU efficiency, the query language must be declarative (SQL or MDX), or at least a vector (J, K). The query should only contain implicit loops, allowing for optimization.
 -->
