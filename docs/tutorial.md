---
slug: /tutorial
sidebar_label: 'Advanced Tutorial'
title: 'Advanced Tutorial'
description: 'Learn how to ingest and query data in ClickHouse using a New York City taxi example dataset.'
sidebar_position: 0.5
keywords: ['clickhouse', 'install', 'tutorial', 'dictionary', 'dictionaries', 'example', 'advanced', 'taxi', 'new york', 'nyc']
show_related_blogs: true
---

# Advanced Tutorial

## Overview {#overview}

Learn how to ingest and query data in ClickHouse using the New York City taxi example dataset. 

### Prerequisites {#prerequisites}

You need access to a running ClickHouse service to complete this tutorial. For instructions, see the [Quick Start](/get-started/quick-start) guide.

<VerticalStepper>

## Create a new table {#create-a-new-table}

The New York City taxi dataset contains details about millions of taxi rides, with columns including tip amount, tolls, payment type, and more. Create a table to store this data.

1. Connect to the SQL console:
    - For ClickHouse Cloud, select a service from the dropdown menu and then select **SQL Console** from the left navigation menu.
    - For self-managed ClickHouse, connect to the SQL console at `https://_hostname_:8443/play`. Check with your ClickHouse administrator for the details.

2. Create the following `trips` table in the `default` database:
    ```sql
    CREATE TABLE trips
    (
        `trip_id` UInt32,
        `vendor_id` Enum8('1' = 1, '2' = 2, '3' = 3, '4' = 4, 'CMT' = 5, 'VTS' = 6, 'DDS' = 7, 'B02512' = 10, 'B02598' = 11, 'B02617' = 12, 'B02682' = 13, 'B02764' = 14, '' = 15),
        `pickup_date` Date,
        `pickup_datetime` DateTime,
        `dropoff_date` Date,
        `dropoff_datetime` DateTime,
        `store_and_fwd_flag` UInt8,
        `rate_code_id` UInt8,
        `pickup_longitude` Float64,
        `pickup_latitude` Float64,
        `dropoff_longitude` Float64,
        `dropoff_latitude` Float64,
        `passenger_count` UInt8,
        `trip_distance` Float64,
        `fare_amount` Float32,
        `extra` Float32,
        `mta_tax` Float32,
        `tip_amount` Float32,
        `tolls_amount` Float32,
        `ehail_fee` Float32,
        `improvement_surcharge` Float32,
        `total_amount` Float32,
        `payment_type` Enum8('UNK' = 0, 'CSH' = 1, 'CRE' = 2, 'NOC' = 3, 'DIS' = 4),
        `trip_type` UInt8,
        `pickup` FixedString(25),
        `dropoff` FixedString(25),
        `cab_type` Enum8('yellow' = 1, 'green' = 2, 'uber' = 3),
        `pickup_nyct2010_gid` Int8,
        `pickup_ctlabel` Float32,
        `pickup_borocode` Int8,
        `pickup_ct2010` String,
        `pickup_boroct2010` String,
        `pickup_cdeligibil` String,
        `pickup_ntacode` FixedString(4),
        `pickup_ntaname` String,
        `pickup_puma` UInt16,
        `dropoff_nyct2010_gid` UInt8,
        `dropoff_ctlabel` Float32,
        `dropoff_borocode` UInt8,
        `dropoff_ct2010` String,
        `dropoff_boroct2010` String,
        `dropoff_cdeligibil` String,
        `dropoff_ntacode` FixedString(4),
        `dropoff_ntaname` String,
        `dropoff_puma` UInt16
    )
    ENGINE = MergeTree
    PARTITION BY toYYYYMM(pickup_date)
    ORDER BY pickup_datetime;
    ```

## Add the dataset {#add-the-dataset}

Now that you've created a table, add the New York City taxi data from CSV files in S3.

1. The following command inserts ~2,000,000 rows into your `trips` table from two different files in S3: `trips_1.tsv.gz` and `trips_2.tsv.gz`:

    ```sql
    INSERT INTO trips
    SELECT * FROM s3(
        'https://datasets-documentation.s3.eu-west-3.amazonaws.com/nyc-taxi/trips_{1..2}.gz',
        'TabSeparatedWithNames', "
        `trip_id` UInt32,
        `vendor_id` Enum8('1' = 1, '2' = 2, '3' = 3, '4' = 4, 'CMT' = 5, 'VTS' = 6, 'DDS' = 7, 'B02512' = 10, 'B02598' = 11, 'B02617' = 12, 'B02682' = 13, 'B02764' = 14, '' = 15),
        `pickup_date` Date,
        `pickup_datetime` DateTime,
        `dropoff_date` Date,
        `dropoff_datetime` DateTime,
        `store_and_fwd_flag` UInt8,
        `rate_code_id` UInt8,
        `pickup_longitude` Float64,
        `pickup_latitude` Float64,
        `dropoff_longitude` Float64,
        `dropoff_latitude` Float64,
        `passenger_count` UInt8,
        `trip_distance` Float64,
        `fare_amount` Float32,
        `extra` Float32,
        `mta_tax` Float32,
        `tip_amount` Float32,
        `tolls_amount` Float32,
        `ehail_fee` Float32,
        `improvement_surcharge` Float32,
        `total_amount` Float32,
        `payment_type` Enum8('UNK' = 0, 'CSH' = 1, 'CRE' = 2, 'NOC' = 3, 'DIS' = 4),
        `trip_type` UInt8,
        `pickup` FixedString(25),
        `dropoff` FixedString(25),
        `cab_type` Enum8('yellow' = 1, 'green' = 2, 'uber' = 3),
        `pickup_nyct2010_gid` Int8,
        `pickup_ctlabel` Float32,
        `pickup_borocode` Int8,
        `pickup_ct2010` String,
        `pickup_boroct2010` String,
        `pickup_cdeligibil` String,
        `pickup_ntacode` FixedString(4),
        `pickup_ntaname` String,
        `pickup_puma` UInt16,
        `dropoff_nyct2010_gid` UInt8,
        `dropoff_ctlabel` Float32,
        `dropoff_borocode` UInt8,
        `dropoff_ct2010` String,
        `dropoff_boroct2010` String,
        `dropoff_cdeligibil` String,
        `dropoff_ntacode` FixedString(4),
        `dropoff_ntaname` String,
        `dropoff_puma` UInt16
    ") SETTINGS input_format_try_infer_datetimes = 0
    ```

2. Wait for the `INSERT` to finish. It might take a moment for the 150 MB of data to be downloaded.

3. When the insert is finished, verify it worked:
    ```sql
    SELECT count() FROM trips
    ```

    This query should return 1,999,657 rows.

## Analyze the data {#analyze-the-data}

Run some queries to analyze the data. Explore the following examples or try your own SQL query. 

- Calculate the average tip amount:
    ```sql
    SELECT round(avg(tip_amount), 2) FROM trips
    ```
    <details>
    <summary>Expected output</summary>
    <p>
    
    ```response
    ┌─round(avg(tip_amount), 2)─┐
    │                      1.68 │
    └───────────────────────────┘
    ```

    </p>
    </details>

- Calculate the average cost based on the number of passengers:
    ```sql
    SELECT
        passenger_count,
        ceil(avg(total_amount),2) AS average_total_amount
    FROM trips
    GROUP BY passenger_count
    ```
    
    <details>
    <summary>Expected output</summary>
    <p>

    The `passenger_count` ranges from 0 to 9:

    ```response
    ┌─passenger_count─┬─average_total_amount─┐
    │               0 │                22.69 │
    │               1 │                15.97 │
    │               2 │                17.15 │
    │               3 │                16.76 │
    │               4 │                17.33 │
    │               5 │                16.35 │
    │               6 │                16.04 │
    │               7 │                 59.8 │
    │               8 │                36.41 │
    │               9 │                 9.81 │
    └─────────────────┴──────────────────────┘
    ```

    </p>
    </details>

- Calculate the daily number of pickups per neighborhood:
    ```sql
    SELECT
        pickup_date,
        pickup_ntaname,
        SUM(1) AS number_of_trips
    FROM trips
    GROUP BY pickup_date, pickup_ntaname
    ORDER BY pickup_date ASC
    ```

    <details>
    <summary>Expected output</summary>
    <p>

    ```response
    ┌─pickup_date─┬─pickup_ntaname───────────────────────────────────────────┬─number_of_trips─┐
    │  2015-07-01 │ Brooklyn Heights-Cobble Hill                             │              13 │
    │  2015-07-01 │ Old Astoria                                              │               5 │
    │  2015-07-01 │ Flushing                                                 │               1 │
    │  2015-07-01 │ Yorkville                                                │             378 │
    │  2015-07-01 │ Gramercy                                                 │             344 │
    │  2015-07-01 │ Fordham South                                            │               2 │
    │  2015-07-01 │ SoHo-TriBeCa-Civic Center-Little Italy                   │             621 │
    │  2015-07-01 │ Park Slope-Gowanus                                       │              29 │
    │  2015-07-01 │ Bushwick South                                           │               5 │
    ```

    </p>
    </details>

- Calculate the length of each trip in minutes, then group the results by trip length:
    ```sql
    SELECT
        avg(tip_amount) AS avg_tip,
        avg(fare_amount) AS avg_fare,
        avg(passenger_count) AS avg_passenger,
        count() AS count,
        truncate(date_diff('second', pickup_datetime, dropoff_datetime)/60) as trip_minutes
    FROM trips
    WHERE trip_minutes > 0
    GROUP BY trip_minutes
    ORDER BY trip_minutes DESC
    ```
    <details>
    <summary>Expected output</summary>
    <p>
    
    ```response
    ┌──────────────avg_tip─┬───────────avg_fare─┬──────avg_passenger─┬──count─┬─trip_minutes─┐
    │   1.9600000381469727 │                  8 │                  1 │      1 │        27511 │
    │                    0 │                 12 │                  2 │      1 │        27500 │
    │    0.542166673981895 │ 19.716666666666665 │ 1.9166666666666667 │     60 │         1439 │
    │    0.902499997522682 │ 11.270625001192093 │            1.95625 │    160 │         1438 │
    │   0.9715789457909146 │ 13.646616541353383 │ 2.0526315789473686 │    133 │         1437 │
    │   0.9682692398245518 │ 14.134615384615385 │  2.076923076923077 │    104 │         1436 │
    │   1.1022105210705808 │ 13.778947368421052 │  2.042105263157895 │     95 │         1435 │
    ```
    </p>
    </details>

- Show the number of pickups in each neighborhood broken down by hour of the day:
    ```sql
    SELECT
        pickup_ntaname,
        toHour(pickup_datetime) as pickup_hour,
        SUM(1) AS pickups
    FROM trips
    WHERE pickup_ntaname != ''
    GROUP BY pickup_ntaname, pickup_hour
    ORDER BY pickup_ntaname, pickup_hour
    ```
    <details>
    <summary>Expected output</summary>
    <p>

    ```response
    ┌─pickup_ntaname───────────────────────────────────────────┬─pickup_hour─┬─pickups─┐
    │ Airport                                                  │           0 │    3509 │
    │ Airport                                                  │           1 │    1184 │
    │ Airport                                                  │           2 │     401 │
    │ Airport                                                  │           3 │     152 │
    │ Airport                                                  │           4 │     213 │
    │ Airport                                                  │           5 │     955 │
    │ Airport                                                  │           6 │    2161 │
    │ Airport                                                  │           7 │    3013 │
    │ Airport                                                  │           8 │    3601 │
    │ Airport                                                  │           9 │    3792 │
    │ Airport                                                  │          10 │    4546 │
    │ Airport                                                  │          11 │    4659 │
    │ Airport                                                  │          12 │    4621 │
    │ Airport                                                  │          13 │    5348 │
    │ Airport                                                  │          14 │    5889 │
    │ Airport                                                  │          15 │    6505 │
    │ Airport                                                  │          16 │    6119 │
    │ Airport                                                  │          17 │    6341 │
    │ Airport                                                  │          18 │    6173 │
    │ Airport                                                  │          19 │    6329 │
    │ Airport                                                  │          20 │    6271 │
    │ Airport                                                  │          21 │    6649 │
    │ Airport                                                  │          22 │    6356 │
    │ Airport                                                  │          23 │    6016 │
    │ Allerton-Pelham Gardens                                  │           4 │       1 │
    │ Allerton-Pelham Gardens                                  │           6 │       1 │
    │ Allerton-Pelham Gardens                                  │           7 │       1 │
    │ Allerton-Pelham Gardens                                  │           9 │       5 │
    │ Allerton-Pelham Gardens                                  │          10 │       3 │
    │ Allerton-Pelham Gardens                                  │          15 │       1 │
    │ Allerton-Pelham Gardens                                  │          20 │       2 │
    │ Allerton-Pelham Gardens                                  │          23 │       1 │
    │ Annadale-Huguenot-Prince's Bay-Eltingville               │          23 │       1 │
    │ Arden Heights                                            │          11 │       1 │
    ```

    </p>
    </details>
    
7. Retrieve  rides to LaGuardia or JFK airports:
    ```sql
    SELECT
        pickup_datetime,
        dropoff_datetime,
        total_amount,
        pickup_nyct2010_gid,
        dropoff_nyct2010_gid,
        CASE
            WHEN dropoff_nyct2010_gid = 138 THEN 'LGA'
            WHEN dropoff_nyct2010_gid = 132 THEN 'JFK'
        END AS airport_code,
        EXTRACT(YEAR FROM pickup_datetime) AS year,
        EXTRACT(DAY FROM pickup_datetime) AS day,
        EXTRACT(HOUR FROM pickup_datetime) AS hour
    FROM trips
    WHERE dropoff_nyct2010_gid IN (132, 138)
    ORDER BY pickup_datetime
    ```

    <details>
    <summary>Expected output</summary>
    <p>

    ```response
    ┌─────pickup_datetime─┬────dropoff_datetime─┬─total_amount─┬─pickup_nyct2010_gid─┬─dropoff_nyct2010_gid─┬─airport_code─┬─year─┬─day─┬─hour─┐
    │ 2015-07-01 00:04:14 │ 2015-07-01 00:15:29 │         13.3 │                 -34 │                  132 │ JFK          │ 2015 │   1 │    0 │
    │ 2015-07-01 00:09:42 │ 2015-07-01 00:12:55 │          6.8 │                  50 │                  138 │ LGA          │ 2015 │   1 │    0 │
    │ 2015-07-01 00:23:04 │ 2015-07-01 00:24:39 │          4.8 │                -125 │                  132 │ JFK          │ 2015 │   1 │    0 │
    │ 2015-07-01 00:27:51 │ 2015-07-01 00:39:02 │        14.72 │                -101 │                  138 │ LGA          │ 2015 │   1 │    0 │
    │ 2015-07-01 00:32:03 │ 2015-07-01 00:55:39 │        39.34 │                  48 │                  138 │ LGA          │ 2015 │   1 │    0 │
    │ 2015-07-01 00:34:12 │ 2015-07-01 00:40:48 │         9.95 │                 -93 │                  132 │ JFK          │ 2015 │   1 │    0 │
    │ 2015-07-01 00:38:26 │ 2015-07-01 00:49:00 │         13.3 │                 -11 │                  138 │ LGA          │ 2015 │   1 │    0 │
    │ 2015-07-01 00:41:48 │ 2015-07-01 00:44:45 │          6.3 │                 -94 │                  132 │ JFK          │ 2015 │   1 │    0 │
    │ 2015-07-01 01:06:18 │ 2015-07-01 01:14:43 │        11.76 │                  37 │                  132 │ JFK          │ 2015 │   1 │    1 │
    ```

    </p>
    </details>

## Create a dictionary {#create-a-dictionary}

A dictionary is a mapping of key-value pairs stored in memory. For details, see [Dictionaries](/sql-reference/dictionaries/index.md) 

Create a dictionary associated with a table in your ClickHouse service.
The table and dictionary are based on a CSV file that contains a row for each neighborhood in New York City. 

The neighborhoods are mapped to the names of the five New York City boroughs (Bronx, Brooklyn, Manhattan, Queens and Staten Island), as well as Newark Airport (EWR).

Here's an excerpt from the CSV file you're using in table format. The `LocationID` column in the file maps to the `pickup_nyct2010_gid` and `dropoff_nyct2010_gid` columns in your `trips` table:

  | LocationID      | Borough |  Zone      | service_zone |
  | ----------- | ----------- |   ----------- | ----------- |
  | 1      | EWR       |  Newark Airport   | EWR        |
  | 2    |   Queens     |   Jamaica Bay   |      Boro Zone   |
  | 3   |   Bronx     |  Allerton/Pelham Gardens    |    Boro Zone     |
  | 4     |    Manhattan    |    Alphabet City  |     Yellow Zone    |
  | 5     |  Staten Island      |   Arden Heights   |    Boro Zone     |

1. Run the following SQL command, which creates a dictionary named `taxi_zone_dictionary` and populates the dictionary from the CSV file in S3. The URL for the file is `https://datasets-documentation.s3.eu-west-3.amazonaws.com/nyc-taxi/taxi_zone_lookup.csv`. 
  ```sql
  CREATE DICTIONARY taxi_zone_dictionary
  (
    `LocationID` UInt16 DEFAULT 0,
    `Borough` String,
    `Zone` String,
    `service_zone` String
  )
  PRIMARY KEY LocationID
  SOURCE(HTTP(URL 'https://datasets-documentation.s3.eu-west-3.amazonaws.com/nyc-taxi/taxi_zone_lookup.csv' FORMAT 'CSVWithNames'))
  LIFETIME(MIN 0 MAX 0)
  LAYOUT(HASHED_ARRAY())
  ```

  :::note
  Setting `LIFETIME` to 0 disables automatic updates to avoid unnecessary traffic to our S3 bucket. In other cases, you might configure it differently. For details, see [Refreshing dictionary data using LIFETIME](/sql-reference/dictionaries#refreshing-dictionary-data-using-lifetime).
  :::

3. Verify it worked. The following should return 265 rows, or one row for each neighborhood:
    ```sql
    SELECT * FROM taxi_zone_dictionary
    ```

4. Use the `dictGet` function ([or its variations](./sql-reference/functions/ext-dict-functions.md)) to retrieve a value from a dictionary. You pass in the name of the dictionary, the value you want, and the key (which in our example is the `LocationID` column of `taxi_zone_dictionary`).

    For example, the following query returns the `Borough` whose `LocationID` is 132, which corresponds to JFK airport):
    ```sql
    SELECT dictGet('taxi_zone_dictionary', 'Borough', 132)
    ```

    JFK is in Queens. Notice the time to retrieve the value is essentially 0:
    ```response
    ┌─dictGet('taxi_zone_dictionary', 'Borough', 132)─┐
    │ Queens                                          │
    └─────────────────────────────────────────────────┘

    1 rows in set. Elapsed: 0.004 sec.
    ```

5. Use the `dictHas` function to see if a key is present in the dictionary. For example, the following query returns `1` (which is "true" in ClickHouse):
    ```sql
    SELECT dictHas('taxi_zone_dictionary', 132)
    ```

6. The following query returns 0 because 4567 is not a value of `LocationID` in the dictionary:
    ```sql
    SELECT dictHas('taxi_zone_dictionary', 4567)
    ```

7. Use the `dictGet` function to retrieve a borough's name in a query. For example:
    ```sql
    SELECT
        count(1) AS total,
        dictGetOrDefault('taxi_zone_dictionary','Borough', toUInt64(pickup_nyct2010_gid), 'Unknown') AS borough_name
    FROM trips
    WHERE dropoff_nyct2010_gid = 132 OR dropoff_nyct2010_gid = 138
    GROUP BY borough_name
    ORDER BY total DESC
    ```

    This query sums up the number of taxi rides per borough that end at either the LaGuardia or JFK airport. The result looks like the following, and notice there are quite a few trips where the pickup neighborhood is unknown:
    ```response
    ┌─total─┬─borough_name──┐
    │ 23683 │ Unknown       │
    │  7053 │ Manhattan     │
    │  6828 │ Brooklyn      │
    │  4458 │ Queens        │
    │  2670 │ Bronx         │
    │   554 │ Staten Island │
    │    53 │ EWR           │
    └───────┴───────────────┘

    7 rows in set. Elapsed: 0.019 sec. Processed 2.00 million rows, 4.00 MB (105.70 million rows/s., 211.40 MB/s.)
    ```

## Perform a join {#perform-a-join}

Write some queries that join the `taxi_zone_dictionary` with your `trips` table.

1. Start with a simple `JOIN` that acts similarly to the previous airport query above:
    ```sql
    SELECT
        count(1) AS total,
        Borough
    FROM trips
    JOIN taxi_zone_dictionary ON toUInt64(trips.pickup_nyct2010_gid) = taxi_zone_dictionary.LocationID
    WHERE dropoff_nyct2010_gid = 132 OR dropoff_nyct2010_gid = 138
    GROUP BY Borough
    ORDER BY total DESC
    ```

    The response looks is identical to the `dictGet` query:
    ```response
    ┌─total─┬─Borough───────┐
    │  7053 │ Manhattan     │
    │  6828 │ Brooklyn      │
    │  4458 │ Queens        │
    │  2670 │ Bronx         │
    │   554 │ Staten Island │
    │    53 │ EWR           │
    └───────┴───────────────┘

    6 rows in set. Elapsed: 0.034 sec. Processed 2.00 million rows, 4.00 MB (59.14 million rows/s., 118.29 MB/s.)
    ```

    :::note
    Notice the output of the above `JOIN` query is the same as the query before it that used `dictGetOrDefault` (except that the `Unknown` values are not included). Behind the scenes, ClickHouse is actually calling the `dictGet` function for the `taxi_zone_dictionary` dictionary, but the `JOIN` syntax is more familiar for SQL developers.
    :::

2. This query returns rows for the the 1000 trips with the highest tip amount, then performs an inner join of each row with the dictionary:
    ```sql
    SELECT *
    FROM trips
    JOIN taxi_zone_dictionary
        ON trips.dropoff_nyct2010_gid = taxi_zone_dictionary.LocationID
    WHERE tip_amount > 0
    ORDER BY tip_amount DESC
    LIMIT 1000
    ```
        :::note
        Generally, we avoid using `SELECT *` often in ClickHouse. You should only retrieve the columns you actually need. However, this query is slower for the purposes of the example.
        :::

</VerticalStepper>

## Next steps {#next-steps}

Learn more about ClickHouse with the following documentation:

- [Introduction to Primary Indexes in ClickHouse](./guides/best-practices/sparse-primary-indexes.md): Learn how ClickHouse uses sparse primary indexes to efficiently locate relevant data during queries. 
- [Integrate an external data source](/integrations/index.mdx): Review data source integration options, including files, Kafka, PostgreSQL, data pipelines, and many others.
- [Visualize data in ClickHouse](./integrations/data-visualization/index.md): Connect your favorite UI/BI tool to ClickHouse.
- [SQL Reference](./sql-reference/index.md): Browse the SQL functions available in ClickHouse for transforming, processing and analyzing data.
