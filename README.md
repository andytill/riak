# Riak Timeseries Fork

This repo, and dependent repos are a fork of what would have been Riak TS 1.6 with some additional features that are a work in progress.

The fork does not have a cool name yet.

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
