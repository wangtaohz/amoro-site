---
title: "Using Logstore"
url: flink-using-logstore
aliases:
    - "flink/using-logstore"
menu:
    main:
        parent: Flink
        weight: 500
---
# Using Logstore
Due to the limitations of traditional offline data warehouse architectures in supporting real-time business needs, real-time data warehousing has experienced rapid evolution in recent years. In the architecture of real-time data warehousing, Apache Kafka is often used as the storage system for real-time data. However, this also brings about the issue of data disconnection between offline data warehouses.

Developers often need to pay attention to data stored in HDFS as well as data in Kafka, which increases the complexity of business development. Therefore, Amoro proposes the addition of an optional parameter, "LogStore enabled" (`log-store.enabled`), to the table configuration. This allows for retrieving data with sub-second and minute-level latency by operating on a single table while ensuring the eventual consistency of data from both sources.

## Real-Time data in LogStore
Amoro tables provide two types of storage: FileStore and LogStore. FileStore stores massive full data, while LogStore stores real-time incremental data.

Real-time data can provide second-level data visibility and ensure data consistency without enabling LogStore transactions.

Its underlying storage system can be connected to external message queuing middleware, currently supporting only Kafka and Pulsar.

Users can enable LogStore by configuring the following parameters when creating an Amoro table. For specific configurations, please refer to [LogStore related configurations](../configurations/#logstore-configurations).

## Overview

| Flink      | Kafka    | Pulsar   |
|------------|----------|----------|
| Flink 1.12 | &#x2714; | &#x2714; |
| Flink 1.14 | &#x2714; | &#x2716; |
| Flink 1.15 | &#x2714; | &#x2716; |

Kafka as LogStore Version Description:

| Flink Version | Kafka Versions |
|---------------|  ----------------- |
| 1.12.x        | 0.10.2.\*<br> 0.11.\*<br> 1.\*<br> 2.\*<br> 3.\*            | 
| 1.14.x        | 0.10.2.\*<br> 0.11.\*<br> 1.\*<br> 2.\*<br> 3.\*            | 
| 1.15.x        | 0.10.2.\*<br> 0.11.\*<br> 1.\*<br> 2.\*<br> 3.\*            | 



### Prerequisites for using LogStore

When creating an Amoro table, LogStore needs to be enabled.

- You can create a table after selecting a specific Catalog on the Amoro [Dashboard](http://localhost:1630) - Terminal page

```sql
CREATE TABLE db.log_table (
    id int,
    name string,
    ts timestamp,
    primary key (id)
) using arctic
tblproperties (
"log-store.enabled" = "true",
"log-store.topic"="topic_log_test",
"log-store.address"="localhost:9092"
);
```

- You can also use Flink SQL to create tables in Flink-SQL-Client

```sql
-- First use the use catalog command to switch to the arctic catalog.
CREATE TABLE db.log_table (
    id int,
    name string,
    ts timestamp,
    primary key (id) not enforced
) WITH (
    'log-store.enabled' = 'true',
    'log-store.topic'='topic_log_test',
    'log-store.address'='localhost:9092');
```

### Double write LogStore and FileStore

![Introduce](../images/flink/auto-writer.png)

Amoro Connector writes data to LogStore and ChangeStore at the same time through double-write operations, without opening Kafka transactions to ensure data consistency between the two, because opening transactions will bring a few minutes of delay to downstream tasks (the specific delay time depends on upstream tasks checkpoint interval).

```sql
INSERT INTO db.log_table /*+ OPTIONS('arctic.emit.mode'='log') */
SELECT id, name, ts from sourceTable;
```

> Currently, only the Apache Flink engine implements the dual-write LogStore and FileStore.
