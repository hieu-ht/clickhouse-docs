---
title: Parallel Snapshot In The MySQL ClickPipe
description: Doc for explaining parallel snapshot in the MySQL ClickPipe
slug: /integrations/data-ingestion/clickpipes/mysql/parallel_initial_load
sidebar_label: 'How parallel snapshot works'
---

import snapshot_params from '@site/static/images/integrations/data-ingestion/clickpipes/postgres/snapshot_params.png'
import partition_key from '@site/static/images/integrations/data-ingestion/clickpipes/mysql/partition_key.png'

This document explains parallelized snapshot/initial load in the MySQL ClickPipe works and talks about the snapshot parameters that can be used to control it.

:::info This feature is currently behind a feature flag
Please reach out to us via a support ticket to enable this feature for your ClickHouse organization.
:::

## Overview

Initial load is the first phase of a CDC ClickPipe, where the ClickPipe syncs the historical data of the tables in the source database over to ClickHouse, before then starting CDC. A lot of the times, developers do this in a single-threaded manner.
However, the MySQL ClickPipe can parallelize this process, which can significantly speed up the initial load.

### Partition key column

Once we've enabled the feature flag, you should see the below setting in the ClickPipe table picker (both during creation and editing of a ClickPipe):
<img src={partition_key} alt="Partition key column" />

The MySQL ClickPipe uses a column on your source table to logically partition the source tables. This column is called the **partition key column**. It is used to divide the source table into partitions, which can then be processed in parallel by the ClickPipe.

:::warning
The partition key column must be indexed in the source table to see a good performance boost.
:::

### Logical partitioning

Let's talk about the below settings:

<img src={snapshot_params} alt="Snapshot parameters" />

#### Snapshot number of rows per partition
This setting controls how many rows constitute a partition. The ClickPipe will read the source table in chunks of this size, and each chunk will be processed in parallel. The default value is 100,000 rows per partition.

#### Initial load parallelism
This setting controls how many partitions will be processed in parallel. The default value is 4, which means that the ClickPipe will read 4 partitions of the source table in parallel. This can be increased to speed up the initial load, but it is recommended to keep it to a reasonable value depending on your source instance specs to avoid overwhelming the source database. The ClickPipe will automatically adjust the number of partitions based on the size of the source table and the number of rows per partition.

#### Snapshot number of tables in parallel
Not really related to parallel snapshot, but this setting controls how many tables will be processed in parallel during the initial load. The default value is 1. Note that is on top of the parallelism of the partitions, so if you have 4 partitions and 2 tables, the ClickPipe will read 8 partitions in parallel.

### Monitoring parallel snapshot in MySQL
You can run **SHOW processlist** in MySQL to see the parallel snapshot in action. The ClickPipe will create multiple connections to the source database, each reading a different partition of the source table. If you see **SELECT** queries with different ranges, it means that the ClickPipe is reading the source tables. You can also see the COUNT(*) and the partitioning query in here.

### Limitations
- The snapshot parameters cannot be edited after pipe creation. If you want to change them, you will have to create a new ClickPipe.
- When adding tables to an existing ClickPipe, you cannot change the snapshot parameters. The ClickPipe will use the existing parameters for the new tables.
