---
slug: /guides/developer/mutations
sidebar_label: 'Updating and Deleting Data'
sidebar_position: 1
keywords: ['UPDATE', 'DELETE', 'mutations']
title: 'Updating and deleting ClickHouse data'
description: 'Describes how to perform update and delete operations in ClickHouse'
show_related_blogs: false
---

# Updating and deleting ClickHouse data with mutations

Although ClickHouse is geared toward high volume analytic workloads, it is possible in some situations to modify or 
delete existing data. These operations are labeled "mutations" and are executed using the `ALTER TABLE` command.

:::tip
If you need to perform frequent updates, consider using [deduplication](../developer/deduplication.md) in ClickHouse, which allows you to update 
and/or delete rows without generating a mutation event. Alternatively, use [lightweight updates](/docs/sql-reference/statements/update)
or [lightweight deletes](/guides/developer/lightweight-delete)
:::

## Updating data {#updating-data}

Use the `ALTER TABLE...UPDATE` command to update rows in a table:

```sql
ALTER TABLE [<database>.]<table> UPDATE <column> = <expression> WHERE <filter_expr>
```

`<expression>` is the new value for the column where the `<filter_expr>` is satisfied.  The `<expression>` must be the same datatype as the column or be convertible to the same datatype using the `CAST` operator.  The `<filter_expr>` should return a `UInt8` (zero or non-zero) value for each row of the data.  Multiple `UPDATE <column>` statements can be combined in a single `ALTER TABLE` command separated by commas.

**Examples**:

 1.  A mutation like this allows updating replacing `visitor_ids` with new ones using a dictionary lookup:

     ```sql
     ALTER TABLE website.clicks
     UPDATE visitor_id = getDict('visitors', 'new_visitor_id', visitor_id)
     WHERE visit_date < '2022-01-01'
     ```

2.   Modifying multiple values in one command can be more efficient than multiple commands:

     ```sql
     ALTER TABLE website.clicks
     UPDATE url = substring(url, position(url, '://') + 3), visitor_id = new_visit_id
     WHERE visit_date < '2022-01-01'
     ```

3.  Mutations can be executed `ON CLUSTER` for sharded tables:

     ```sql
     ALTER TABLE clicks ON CLUSTER main_cluster
     UPDATE click_count = click_count / 2
     WHERE visitor_id ILIKE '%robot%'
     ```

:::note
It is not possible to update columns that are part of the primary or sorting key.
:::

## Deleting data {#deleting-data}

Use the `ALTER TABLE` command to delete rows:

```sql
ALTER TABLE [<database>.]<table> DELETE WHERE <filter_expr>
```

The `<filter_expr>` should return a UInt8 value for each row of data.

**Examples**

1. Delete any records where a column is in an array of values:
    ```sql
    ALTER TABLE website.clicks DELETE WHERE visitor_id in (253, 1002, 4277)
    ```

2.  What does this query alter?
    ```sql
    ALTER TABLE clicks ON CLUSTER main_cluster DELETE WHERE visit_date < '2022-01-02 15:00:00' AND page_id = '573'
    ```

:::note
To delete all of the data in a table, it is more efficient to use the command `TRUNCATE TABLE [<database].]<table>` command.  This command can also be executed `ON CLUSTER`.
:::

View the [`DELETE` statement](/sql-reference/statements/delete.md) docs page for more details.

## Lightweight deletes {#lightweight-deletes}

Another option for deleting rows it to use the `DELETE FROM` command, which is referred to as a **lightweight delete**. The deleted rows are marked as deleted immediately and will be automatically filtered out of all subsequent queries, so you do not have to wait for a merging of parts or use the `FINAL` keyword. Cleanup of data happens asynchronously in the background.

``` sql
DELETE FROM [db.]table [ON CLUSTER cluster] [WHERE expr]
```

For example, the following query deletes all rows from the `hits` table where the `Title` column contains the text `hello`:

```sql
DELETE FROM hits WHERE Title LIKE '%hello%';
```

A few notes about lightweight deletes:
- This feature is only available for the `MergeTree` table engine family.
- Lightweight deletes are asynchronous by default. Set `mutations_sync` equal to 1 to wait for one replica to process the statement, and set `mutations_sync` to 2 to wait for all replicas.
