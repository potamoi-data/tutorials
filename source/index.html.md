---
title: Ecommerce use case

language_tabs: # must be one of https://git.io/vQNgJ
# markdown is for output of sql queries. TODO: tab should display "output".
  - sql
  - markdown
  
toc_footers:
  - <a href='https://github.com/slatedocs/slate'>Documentation Powered by Slate</a>

search: true

code_clipboard: true
---

# Introduction

> Ensure that you start clickhouse client with multi-line and multi-statement enabled using this bash command.

```bash
clickhouse-client -mn
```

> Create a separate database for this tutorial.

```sql
CREATE DATABASE ecommerce;
USE ecommerce; 
```

In this tutorial, you will 

* define the schema
* ingest the data
* analyze the data
* tune clickhouse to optimize performance

The [Data](https://www.kaggle.com/retailrocket/ecommerce-dataset) contains the clickstream on an ecommerce website. Read the full description of the data before starting.

You must define your use case before you start this exercise. Let's analyze the data in the following ways:

1. Get funnel of view -> add to cart -> transact  
    a. for an item  
    b. for items with a certain property value  
    c. for a category  
2. When/where is conversion the best?  
    a. Which Time of day, day of week, dates of year  
    b. Which category  
    c. Which properties (single key value pair)  
3. What is the trend of  
    a. Views, addition to cart, transactions  
    b. Conversion %  

# Create tables in ch

> Use this to create your tables. We shall dive deeper into some of the lines in further sections  

```sql
CREATE TABLE IF NOT EXISTS ecommerce.events 
(
    timestamp       DateTime64(3),
    visitorid       UInt32,
    event           LowCardinality(String),
    itemid          UInt32,
    transactionid   UInt32
)
ENGINE = MergeTree
PARTITION BY toYYYYMMDD(timestamp)
ORDER BY (event, itemid, timestamp)
PRIMARY KEY (event, itemid);

CREATE TABLE IF NOT EXISTS ecommerce.item_properties
(
    timestamp       DateTime64(3),
    itemid          UInt32,
    property        LowCardinality(String),
    value           String,
    version         UInt32 MATERIALIZED 9999999999 - toUnixTimestamp(timestamp) Codec(Delta, LZ4)
)
ENGINE = ReplacingMergeTree(version)
ORDER BY (itemid, property, value)
PRIMARY KEY (itemid, property);

CREATE TABLE IF NOT EXISTS ecommerce.category_tree
(
    categoryid      UInt16 Codec(T64),
    parentid        UInt16 Codec(T64)
)
ENGINE = MergeTree
ORDER BY (categoryid);
```

When creating a table in clickhouse, the following decisions are important because clickhouse provides you with a lot of options for each.

1. which table engine to use?
2. what datatypes to use?
3. what sorting key to use? should your primary key be different from the sorting key?
4. can you reduce storage size by from using a compression codec different from the default one for certain columns?

Like all other data warehouses, this decision depends on the shape of data and what kind of queries your use case demands. The data explorer section on the dataset page will come in handy in this.


## Picking the right table engine

Clickhouse provides a genuine variety of table engines unlike other SQL databases. 

### MergeTree family
> Highlighting the relevant lines from the creation sql snippet

```sql
CREATE TABLE IF NOT EXISTS ecommerce.events 
...
ENGINE = MergeTree
```

Pre-requisite: In most cases you would be interested in the MergeTree family of table engines. Go through all engines in [docs](https://clickhouse.tech/docs/en/engines/table-engines/mergetree-family/).

For each table consider if a special MergeTree would suit your needs better. For example, are there rows in your table that are an updated version of another row? If yes then ReplacingMergeTree suits your need. When you insert a new row in it, the earlier version of the row is deleted along with inserting the new row.

HW: Read distributed & replicating merge trees.

### ReplacingMergeTree

```sql
CREATE TABLE IF NOT EXISTS ecommerce.item_properties
...
    version         UInt32 MATERIALIZED 9999999999 - toUnixTimestamp(timestamp) Codec(Delta, LZ4)
...
ENGINE = ReplacingMergeTree(version)
```

Often, data is stored in a system of record or saas app like salesforce or freshdesk or your own SQL database. The common practice is to take snapshot of such data and ingest it into your data warehouse for analytics, periodically. The `item_properties` table is like that. New rows are inserted into for every `(item, property, snapshot_time)` irrespective of whether the property value changed or not. Such tables can grow very quickly because of duplicate rows. You can use ReplacingMergeTree to ingest/store such data. Clickhouse will not increase the table size this way. 

We could have used `timestamp` column as the version argument to the ReplacingMergeTree. Why did we need to create a separate `version` column? Because when a new row is inserted in the table with the exact same values as the previous snapshot we want to keep the earlier version and discard the new one. This ensures when a property is updated in the system of record, the warehouse stores the closest snapshot time of the change. It can enable point-in-time analytical use cases like how many views did we get for an item with a given property say, a month ago.


## Column data type
> Toggle to markdown tab to see output of this query. Notice the high compression ratio for integer, datetime and low cardinality data types.

```sql
SELECT
    table,
    name,
    sum(data_compressed_bytes) AS comp,
    sum(data_uncompressed_bytes) AS uncomp,
    round((uncomp / comp), 2) AS comp_ratio
FROM system.columns
WHERE database = 'ecommerce' AND table = 'item_properties'
GROUP BY table, name
ORDER BY table, name;
```

```markdown
table | name | comp | uncomp | comp_ratio
----- | ---- | ---- | ------ | ----------
item_properties | itemid | 2117248 | 51114948 | 24.14
item_properties | property | 6581328 | 25586875 | 3.89
item_properties | timestamp | 32696898 | 102229896 | 3.13
item_properties | value | 104781287 | 203559039 | 1.94
item_properties | version | 22231770 | 51114948 | 2.3
```

UInt, Float, String and DateTime will fulfill most of your needs but here are some other extremely useful types:

* LowCardinality: When the cardinality of your column is less than 10k then use this. Prefer it over Enum when working with strings. It provides more flexibility in use and often reveals the same or higher efficiency
* Array, Tuple, Map: You will find it very handy when transforming data using views for your common use cases

<aside class="notice">
Clickhouse compresses the data in each column based on the data type, out-of-the-box. Clickhouse has system tables to help you analyze this. See compression ratios on the right. 
</aside>

Note that some data types are case sensitive. For eg, `date` and `Date` both are valid types but that is not true for some data types. You can query a [system table](https://clickhouse.tech/docs/en/operations/system-tables/data_type_families/#system_tables-data_type_families) to check this for any data type.


## Sorting and primary key

> Highlighting the relevant lines from the creation sql snippet

```sql
CREATE TABLE IF NOT EXISTS ecommerce.events 
...
PARTITION BY toYYYYMMDD(timestamp)
ORDER BY (event, itemid, timestamp)
PRIMARY KEY (event, itemid);

CREATE TABLE IF NOT EXISTS ecommerce.item_properties
...
ORDER BY (itemid, property, value)
PRIMARY KEY (itemid, property);
```

Pre-requisite: Read this [primer in docs](https://clickhouse.tech/docs/en/engines/table-engines/mergetree-family/mergetree) on how primary and sorting keys are used. 

Quoting highlights from the docs:

* It is possible to specify a primary key (an expression with values that are written in the index file for each mark) that is different from the sorting key (an expression for sorting the rows in data parts). In this case the primary key expression tuple must be a prefix of the sorting key expression tuple.
* A long primary key will negatively affect the insert performance and memory consumption, but not SELECT queries.

We have added `timestamp` column to the sorting key in `events` table and `value` column in `item_properties` table because we may filter on those columns in certain queries. But because both these are high cardinality columns that can impact resource utilization. So we can omit them from the primary key.

## Compression codecs

> I was able to improve compression ratio for only these columns through experimentation. Clickhouse picked the best algorithm for the rest out-of-the-box.

```sql
CREATE TABLE IF NOT EXISTS ecommerce.item_properties
...
    version         UInt32 MATERIALIZED 9999999999 - toUnixTimestamp(timestamp) Codec(Delta, LZ4)
...

CREATE TABLE IF NOT EXISTS ecommerce.category_tree
...
    categoryid      UInt16 Codec(T64),
    parentid        UInt16 Codec(T64)
...
```

Pre-requisite: Read this [primer in docs](https://clickhouse.tech/docs/en/sql-reference/statements/create/table/#codecs).

Try different codecs and decide which one to use based on the compression ratio. The sql snippet in [column data type section](#column-data-type) can be used to check compression ratio before and after a change. 


# Insert data into tables

> These are bash commands.

```bash
clickhouse-client --query "INSERT INTO ecommerce.events FORMAT CSVWithNames" < events.csv
clickhouse-client --query "INSERT INTO ecommerce.item_properties (timestamp, itemid, property, value) FORMAT CSVWithNames" < item_properties_part1.csv
clickhouse-client --query "INSERT INTO ecommerce.item_properties (timestamp, itemid, property, value) FORMAT CSVWithNames" < item_properties_part2.csv
clickhouse-client --query "INSERT INTO ecommerce.category_tree FORMAT CSVWithNames" < category_tree.csv
```

It takes upto 10s for clickhouse to ingest all the data. The `FORMAT CSVWithNames` phrase was needed because the first row of the CSV files contained the column names


# Create views for queries

## Get latest/current properties of item

```sql
CREATE MATERIALIZED VIEW ecommerce.item_properties_latest_mv
ENGINE = AggregatingMergeTree()
ORDER BY (itemid, property)
POPULATE AS 
    SELECT 
        argMax(value, timestamp) as latest_value,
        max(timestamp) as latest_timestamp,
        itemid,
        property
    FROM ecommerce.item_properties
    GROUP BY itemid, property;
```

### AggregatingMergeTree table engine
ClickHouse merges all rows with the same sorting key with a single aggregated row (within one data part). Atleast one column must be of an AggregateFunction data type. In the SQL snippet latest_value & latest_timestamp columns are of that type.

HW: Read docs for [AggregateMergeTree](https://clickhouse.tech/docs/en/engines/table-engines/mergetree-family/aggregatingmergetree/).

### AggregateFunction data type & aggregate functions
Pre-requisite: Read docs for [AggregateFunction data type](https://clickhouse.tech/docs/en/sql-reference/data-types/aggregatefunction/) and [aggregate functions](https://clickhouse.tech/docs/en/sql-reference/aggregate-functions/) that return data of this type.

For ex, `max(timestamp)` returns data of type `AggregateFunction(max, DateTime64(3))`.

### -State & -Merge combinators of aggregate functions
```sql
CREATE MATERIALIZED VIEW ecommerce.item_properties_latest_mv_state
ENGINE = AggregatingMergeTree()
ORDER BY (itemid, property)
POPULATE AS 
    SELECT 
        argMaxState(value, timestamp) as latest_value,
        maxState(timestamp) as latest_timestamp,
        itemid,
        property
    FROM ecommerce.item_properties
    GROUP BY itemid, property;

CREATE VIEW ecommerce.item_properties_latest AS
SELECT 
    argMaxMerge(latest_value) as value,
    maxMerge(latest_timestamp) as timestamp,
    itemid,
    property
FROM ecommerce.item_properties_latest_mv_state
GROUP BY itemid, property;

CREATE VIEW ecommerce.item_category_latest AS
SELECT 
    argMaxMerge(latest_value) as categoryid,
    itemid
FROM ecommerce.item_properties_latest_mv_state
WHERE property = 'categoryid'
GROUP BY itemid;
```

An aggregate function can have a combinator which changes the way it works. For example, the `State` suffix is used to represent intermediate state of applying a function. In this snippet, the latest_timestamp column is stored in binary form. The intermediate state is stored using a data structure that fits the function. For example a hash function is used for uniq(). 

The intermediate state can be materialized such that further computations on it can be run very efficiently & flexibly. Multiple views may be created on top of the materialized view as demonstrated in this snippet. The `Merge` suffix merges the intermediate output into final values.

HW: Read `If`, `State`, `Merge`, `MergeState` in [docs](https://clickhouse.tech/docs/en/sql-reference/aggregate-functions/combinators).

### SimpleAggregateFunction & -Simple combinator
Pre-requisite: Read docs for [simple aggregate functions](https://clickhouse.tech/docs/en/sql-reference/data-types/simpleaggregatefunction/).

You can replace `max()` with `maxSimple()`, `maxState()` with `maxSimpleState()` in the above snippets and it will make the storage and queries slightly more efficient. 

HW: You can refer to altinity (a managed clickhouse provider) [knowledge base](https://kb.altinity.com/altinity-kb-queries-and-syntax/simplestateif-or-ifstate-for-simple-aggregate-functions#q-what-is-simpleaggregatefunction-are-there-advantages-to-use-it-instead-of-aggregatefunction-in-aggregatingmergetree) to know more.

## Get all properties in one row using Array & Map functions

```sql
SET allow_experimental_map_type = 1;
CREATE MATERIALIZED VIEW IF NOT EXISTS ecommerce.items_latest
ENGINE = Join(ANY, INNER, id)
POPULATE AS 
    SELECT 
        CAST ((keys, values), 'Map(String, String)') as properties,
        updated_at,
        itemid as id
    FROM (
        SELECT
            groupArray(property) as keys,
            groupArray(value) as values,
            max(timestamp) as updated_at,
            itemid
        FROM ecommerce.item_properties_latest
        GROUP BY itemid
    )
    ORDER BY id;
```

### Query settings
Since `map` is an experimental feature, we need to enable a setting to use it. You can set settings at different levels: cluster wide, per user, per session or per query.

### groupArray(): merge multiple rows into an array
`SELECT groupArray(col_x) ... GROUP BY col_y` does this - for all rows that have the same value in column y, extract all values in column x and put them in an array. 

### arrayJoin(): unpack an array into multiple rows
It is useful to also learn about the function that can do the opposite of `groupArray()`. `arrayJoin(arr)` returns a table in which the first row is arr[0], second row is arr[1], and so on.

### arrayMap(): merge multiple arrays into one array
`arrayMap(<lambda_function>, arr1, arr2)` returns a list where the first element is <lambda_function> applied to arr1[0] & arr2[0], second element is <lambda_function> applied to arr1[1] & arr2[2] & so on. 

### Join engine
It is used to create a table/view that can be joined with another easily. If a table is expected to be joined with several other tables/views then you can use this engine to define the join type, strictness & columns at one place rather than defining it in every join clause.

HW: Read [docs](https://clickhouse.tech/docs/en/engines/table-engines/special/join/).


# Query: Get funnel of conversion

Let's explore a few approaches of filtering certain rows of a table and aggregating them. In typical data bases/warehouses you only have basic functions like the ones used in approach #1. Clickhouse provides you powerful functions like `retention()` and `sequenceCount()` that enable you to analyze click stream data with much simpler SQL queries. You will find that complexity goes down as we go from approach #1 to #3.

## Approach #1: Using SummingMergeTree, Array & Map functions

```sql
CREATE MATERIALIZED VIEW IF NOT EXISTS ecommerce.agg_events_by_item_mv
(
    `itemid` UInt32,
    `event` LowCardinality(String),
    `num_events` UInt32,
    `num_users` UInt32
)
ENGINE = SummingMergeTree((num_events, num_users))
ORDER BY (itemid, event) POPULATE AS
SELECT
    itemid,
    event,
    count(timestamp) AS num_events,
    uniq(visitorid) AS num_users
FROM ecommerce.events
GROUP BY itemid, event;

SET allow_experimental_map_type = 1;
create view agg_events_by_item_in_map as
select
    CAST ((num_events_arr.event, num_events_arr.num_events), 'Map(String, UInt32)') as num_events_map,
    itemid
from (
select 
    groupArray((event, num_events)) as num_events_arr,
    groupArray((event, num_users)) as num_users_arr,
    itemid
from ecommerce.agg_events_by_item_mv
group by itemid
)
order by itemid;
```

### Step 1 
Get sum of events and users by event type and item

### Step 2
Put it in a map like {'a': 1,'b': 9,'c': 0} 

### Step 3
Compute conversion ratios using array elements & split array elements into rows. Implement this as HW. It's SQL snippet is not shared.

## Approach #2: Using retention() 

```sql
create view events_funnel_using_retention as
select
    itemid,
    sum(r[1]) AS r1,
    sum(r[2]) AS r2,
    sum(r[3]) AS r3
from (
select 
    itemid,
    visitorid,
    retention(event = 'view', event = 'addtocart', event = 'transaction') as r
from ecommerce.events
group by itemid, visitorid
order by itemid
)
group by itemid
-- order by r3 desc
-- limit 5
;
```

Drawback: it doesn't take time into consideration. If you bought an item on day N, and you add it to your cart again on day N+1 then it will be contribute to increasing the retention score, whereas it shouldn't because these two events were part of two different user journeys. The first one finished when the item was bought on day N. Apprack #3 can handle this better.

## Approach #3: using sequenceCount()

```sql
create view events_funnel as
select
    itemid,
    visitorid,
    countIf(timestamp, event='view') as num_view,
    sequenceCount('(?1)(?2)')(
        toDateTime(timestamp), 
        event='view', 
        event='addtocart'
    ) as num_addtocart,
    sequenceCount('(?1)(?2)(?3)')(
        toDateTime(timestamp), 
        event='view', 
        event='addtocart', 
        event='transaction'
    ) as num_transaction
from ecommerce.events
group by itemid, visitorid
-- order by num_transaction desc, num_addtocart desc, num_view desc
-- limit 5
;
```

Note: Param #1 of sequenceCount() can't be a column of data type `datetime64`. It has to be `datetime`.


# Query: Get funnel by item, category etc

We can now query the `events_funnel` view to answer all out queries.

## Aggregate to get by item

```sql
create view funnel_by_item as
select
    itemid,
    round(sum_addtocart/sum_view, 2) as view_to_cart,
    round(sum_transaction/sum_addtocart, 2) as cart_to_trxn
from (
select    
    itemid,
    sum(num_view) as sum_view,
    sum(num_addtocart) as sum_addtocart,
    sum(num_transaction) as sum_transaction
from events_funnel
group by itemid
)
-- order by cart_to_trxn desc, view_to_cart desc
-- limit 5
;
```

## Join with category view to get by category

```sql
create view funnel_by_category as
select 
    categoryid,
    round(sum_addtocart/sum_view, 2) as view_to_cart,
    round(sum_transaction/sum_addtocart, 2) as cart_to_trxn
from (
select    
    items_category_latest.categoryid as categoryid,
    sum(num_view) as sum_view,
    sum(num_addtocart) as sum_addtocart,
    sum(num_transaction) as sum_transaction
from events_funnel right outer join items_category_latest
on events_funnel.itemid = items_category_latest.id 
group by categoryid
)
-- order by cart_to_trxn desc, view_to_cart desc
-- limit 5
;
```

Join with table containing latest category definition of each item.

