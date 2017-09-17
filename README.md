# MinhDB

> **A Comrades Paper Blanket**
>
> New books, old books,
> the leaves all piled together.
>
> A paper blanket
> is better than no blanket.
>
> You who sleep like princes,
> sheltered from the cold,
>
> Do you know how many men in prison
> cannot sleep all night?
>
> \- Ho Chi Minh

This repo, and dependent repos are a fork of what would have been Riak TS 1.6 with some additional features that are a work in progress. It is a sandbox for features that were discussed by the Riak TS team but were never implemented before Basho went into receivership.

This fork is not production ready. On-disk formats are not compatible with Riak TS 1.5, on-disk formats may change in the future and rolling upgrade is not supported.

### Differences to Riak TS 1.5

#### Riak TS 1.6 Features

- `now()` function supported in select and where clauses
- arithmetic in where clause
- group by time
- `IN` keyword

#### HanoiDB Backend

The backend for this fork is HanoiDB and not LevelDB that was used in Basho's Riak TS. The reason for this is that the complexity of LevelDB meant that a dedicated engineer was required to make changes to it. For this fork, I wanted a backend that could be changed much more quickly to allow experimentation.

#### Multiple Partition Key Values Allowed in Queries

Riak TS 1.5 required that a field in the partition key was specified exactly.

Note that column `a` requires an exact predicate `a = 'mydevice'`, it is not possible to specify a range or multiple values. This is because the partition a "start key" value that it can hash to find which partition the rows are in the cluster.

```sql
CREATE TABLE mytable(
a VARCHAR NOT NULL,
b TIMESTAMP NOT NULL,
PRIMARY KEY((a,QUANTUM(b,1,'s')), a,b)
);

SELECT * FROM mytable WHERE a = 'mydevice' AND b > 1000 AND b < 2000;
```

In this fork, several values can be specified.

```sql
-- the IN keyword, or OR can be used, but **must** be contained in brackets
SELECT * FROM mytable WHERE (a = 'mydevice' OR a = 'myotherdevice') AND b > 1000 AND b < 2000;
SELECT * FROM mytable WHERE a IN ('mydevice', 'myotherdevice') AND b > 1000 AND b < 2000;
```

Running `EXPLAIN` on the original query shows that one sub query will be executed.

```sql
riak-shell(41)>EXPLAIN SELECT * FROM mytable WHERE a = 'mydevice' AND b > 1000 AND b < 2000;
+--------+----------------------------------------------------+------------------------+-------------------+------------------------+-----------------+------+
|Subquery|                   Coverage Plan                    |  Range Scan Start Key  |Is Start Inclusive?|   Range Scan End Key   |Is End Inclusive?|Filter|
+--------+----------------------------------------------------+------------------------+-------------------+------------------------+-----------------+------+
|   1    |dev1@127.0.0.1/4, dev1@127.0.0.1/5, dev1@127.0.0.1/6|a = 'mydevice', b = 1001|       false       |a = 'mydevice', b = 2000|      false      |      |
+--------+----------------------------------------------------+------------------------+-------------------+------------------------+-----------------+------+
```

Running `EXPLAIN` on the query with multiple partition key values shows that one sub query will be executed per

```sql
riak-shell(43)>EXPLAIN SELECT * FROM mytable WHERE a IN ('mydevice','myotherdevice') AND b > 1000 AND b < 2000;
+--------+-------------------------------------------------------+-----------------------------+-------------------+-----------------------------+-----------------+------+
|Subquery|                     Coverage Plan                     |    Range Scan Start Key     |Is Start Inclusive?|     Range Scan End Key      |Is End Inclusive?|Filter|
+--------+-------------------------------------------------------+-----------------------------+-------------------+-----------------------------+-----------------+------+
|   1    |dev1@127.0.0.1/56, dev1@127.0.0.1/57, dev1@127.0.0.1/58|a = 'myotherdevice', b = 1001|       false       |a = 'myotherdevice', b = 2000|      false      |      |
|   2    | dev1@127.0.0.1/4, dev1@127.0.0.1/5, dev1@127.0.0.1/6  |  a = 'mydevice', b = 1001   |       false       |  a = 'mydevice', b = 2000   |      false      |      |
+--------+-------------------------------------------------------+-----------------------------+-------------------+-----------------------------+-----------------+------+
```

#### Execute the `SELECT`, `LIMIT`, `GROUP BY` and `ORDER BY` clause on the vnode

> Send the query to the data, not the data to the query

Queries are now executed as follows:

- SQL is parsed and compiled into sub query AST.
- Sub query AST is sent to the vnode responsible for the keys.
- The vnode compiles sub query AST to erlang funs that generate the result for each row.
- The vnode executes the select clause funs over each row.
- If the query is an aggregate or group by then the results are aggregated or grouped.
- If the query has an order by clause then the rows are sorted in memory.
- If the query has a limit clause then the results are truncated. If an offset then (offset + limit) rows are returned and the offset is discarded at the coordinator.
- The vnode sends the results to the coordinator. Since the are multiple sub queries the coordinator must merge the results together to get a final result.

By executing the `SELECT` clause at the vnode the work of the query is shared across the cluster instead of at a single point, the coordinator. Depending on the query, the result transmitted across the network may be much smaller than the full row set e.g. if the query is an aggregate, group by or has a limit.

#### Per-Bucket-Type Row Expiry

To specify an expiry time for the rows in a table (bucket type), declare `expiry_secs` as a bucket type property.

```sql
CREATE TABLE mytable(
    a VARCHAR NOT NULL,
    b TIMESTAMP NOT NULL,
    PRIMARY KEY((a,QUANTUM(b,1,'s')), a,b)
) WITH (expiry_secs = 10);
```

This table will return rows that are written for 10 seconds after they are received by the hanoidb backend. After that time the rows will no longer be accessible and will be deleted from disk when compaction occurs. Arithmetic or the `24h` syntax is not supported.