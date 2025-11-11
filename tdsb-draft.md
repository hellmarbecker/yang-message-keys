## Datastore

(to be added after section 4 "Message Broker")

### 1. Time Series Databases

...

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

Unlike other realtime analytics databases, ClickHouse does not rely on partitioning data by timestamp. ClickHouse represents data in the _MergeTree_ format:

A table consists of data parts sorted by primary key.

When data is inserted in a table, separate data parts are created and each of them is lexicographically sorted by primary key. For example, if the primary key is (CounterID, Date), the data in the part is sorted by CounterID, and within each CounterID, it is ordered by Date.

Data belonging to different partitions are separated into different parts. In the background, ClickHouse merges data parts for more efficient storage. Parts belonging to different partitions are not merged. The merge mechanism does not guarantee that all rows with the same primary key will be in the same data part.
