## Datastore

(to be added after section 4 "Message Broker")

### 1. Time Series Databases

Time series databases would usually partition and sort incoming data by timestamp. 

Advantages:

- Fast selection of subsets by time interval
- Built in support for aggregation (rollup) by 

Disadvantages:

### 2. ClickHouse

#### Message Broker Integration

ClickHouse integrates with Message Brokers through Integration Table Engines. 

Here is a DDL example:

```
CREATE TABLE queue (
    timestamp UInt64,
    level String,
    message String
  ) ENGINE = Kafka('localhost:9092', 'topic', 'group1', 'JSONEachRow');
```

Reading (selecting) data through Kafka Table Engine follows Kafka semantics of advancing the offset, so subsequent reads will start at the offset the previous read left off.

It is the responsibility of the data model designer to transfer data to a regular table:

1. Use the engine to create a Kafka consumer and consider it a data stream.
2. Create a table with the desired structure.
3. Create a materialized view that converts data from the engine and puts it into a previously created table.

The message key and partition ID are available as virtual (read only) columns `_key` and `_partition`.

#### Data Model

Unlike other realtime analytics databases, ClickHouse does not (necessarily) rely on partitioning data by timestamp. ClickHouse represents data in the _MergeTree_ format:

A table consists of data parts sorted by primary key.

When data is inserted in a table, separate data parts are created and each of them is lexicographically sorted by primary key. For example, if the primary key is (MessageKey, Date), the data in the part is sorted by MessageKey, and within each MessageKey, it is ordered by Date.

Data belonging to different partitions are separated into different parts. In the background, ClickHouse merges data parts for more efficient storage. Parts belonging to different partitions are not merged. The merge mechanism does not guarantee that all rows with the same primary key will be in the same data part.

Each data part is logically divided into granules. A granule is the smallest indivisible data set that ClickHouse reads when selecting data. ClickHouse does not split rows or values, so each granule always contains an integer number of rows. The first row of a granule is marked with the value of the primary key for the row. For each data part, ClickHouse creates an index file that stores the marks. For each column, whether it's in the primary key or not, ClickHouse also stores the same marks. These marks let you find data directly in column files.

Thus, it is possible to quickly run queries on one or many ranges of the primary key. 


TODO:

- keep history that is not in kafka (because of limited retention or log compaction)
- fill data points to a grid where there are no changes (like continuing the last known state every hour or so)
