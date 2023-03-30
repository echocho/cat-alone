# Some Common Misunderstandings of MySQL


## Misunderstanding 1: Conversion between UTC and non-UTC timezones are one-to-one
In MySQL, temporal values are stored in `timestamp` as UTC values. When inserting and retrieving `timestamp` columns, if session time zone is not in UTC, there will be convertsion between UTC and session time zone.
Due to local time zone changes like Daylight Saving Time, conversions between UTC and non-UTC time zones are not one-to-one in both directions.
For example, two distinct UTC timestamps are not unique in MET timezone.
```sql
mysql> CREATE TABLE tztable (ts TIMESTAMP);
-- insert UTC values
mysql> SET time_zone = 'UTC'; 
mysql> INSERT INTO tztable VALUES
       ('2018-10-28 00:30:00'),
       ('2018-10-28 01:30:00');
mysql> SELECT ts FROM tztable;
+---------------------+
| ts                  |
+---------------------+
| 2018-10-28 00:30:00 |
| 2018-10-28 01:30:00 |
+---------------------+
mysql> SET time_zone = 'MET'; -- retrieve non-UTC values
mysql> SELECT ts FROM tztable;
+---------------------+
| ts                  |
+---------------------+
| 2018-10-28 02:30:00 |
| 2018-10-28 02:30:00 |
+---------------------+

```

Comparison of UTC and non-UTC timezones may yield different results, depending on if it is an indexed or nonindexed lookup.
If there's no usable index on the `timestamp` column, comparison occurs in the session timezone. Specifically, the optimizer 
1. first performs a table scan, 
2. retrieves all `ts` values, 
3. converts timestamp values from UTC to session time zone,
4. compares them with the searched value.  
```sql
mysql> SELECT ts FROM tztable where ts = '2018-10-28 02:30:00';
+---------------------+
| ts                  |
+---------------------+
| 2018-10-28 02:30:00 |
| 2018-10-28 02:30:00 |
+---------------------+
```

If there's a usable index on the `timestamp` column, comparison occur in UTC. The optimizer does the following:
1. first performs an index scan,
2. converts the search value from the session time zone to UTC,
3. compares the result to the UTC index entries
```sql
mysql> ALTER TABLE tstable ADD INDEX (ts);
mysql> SELECT ts FROM tstable
       WHERE ts = '2018-10-28 02:30:00';
+---------------------+
| ts                  |
+---------------------+
| 2018-10-28 02:30:00 |
+---------------------+
```

_Reference_
https://dev.mysql.com/doc/refman/8.0/en/timestamp-lookups.html

## Misunderstanding 2: Comparing dissimilar columns is at no cost
Let's start with a sample table.
```sql
mysql> SELECT * FROM test.teacher;
+--------+-------+-----+-------------+------------+
| id     | name  | age | contract_id | join_on    |
+--------+-------+-----+-------------+------------+
| 100000 | john  |  30 | 100000      | 2023-01-01 |
| 100001 | hanna |  35 | 100200      | 2023-02-01 |
| 100003 | mary  |  70 | 100300      | 2023-03-01 |
| 100004 | peter |  66 | 100400      | 2023-04-01 |
+--------+-------+-----+-------------+------------+
4 rows in set (0.00 sec)

mysql> DESCRIBE test.teacher;
+-------------+-------------+------+-----+---------+-------+
| Field       | Type        | Null | Key | Default | Extra |
+-------------+-------------+------+-----+---------+-------+
| id          | varchar(10) | NO   | PRI | NULL    |       |
| name        | varchar(30) | NO   |     | NULL    |       |
| age         | int(11)     | NO   |     | NULL    |       |
| contract_id | varchar(20) | NO   | MUL | NULL    |       |
| join_on     | date        | YES  |     | NULL    |       |
+-------------+-------------+------+-----+---------+-------+
5 rows in set (0.00 sec)
```
Note the type of `contract_id` is `varchar` as you can see in `DESCRIBE`.
It's not uncommon to see query like `SELECT * FROM test.teacher WHERE contract_id = 100400;`, where it compares a string column to a numeric literal. This, however, prevent use of indexes if values cannot be compared directly without conversion. Think about it, For a given value such as 1 in the numeric column, it might compare equal to any number of values in the string column such as '1', ' 1', '00001', or '01.e1'. This means the potentially all values need to be compared.

We can understand better with `EXPLAIN` statement. Even though there are possible index `contract_id`, it is not used. The value for `row` column is 4 so we know that the optimizer does a full table scan and compares the value one by one.
```sql
mysql> EXPLAIN select * from test.teacher where contract_id = 100400;
+----+-------------+---------+------------+------+---------------+------+---------+------+------+----------+-------------+
| id | select_type | table   | partitions | type | possible_keys | key  | key_len | ref  | rows | filtered | Extra       |
+----+-------------+---------+------------+------+---------------+------+---------+------+------+----------+-------------+
|  1 | SIMPLE      | teacher | NULL       | ALL  | contract_id   | NULL | NULL    | NULL |    4 |   100.00 | Using where |
+----+-------------+---------+------------+------+---------------+------+---------+------+------+----------+-------------+
1 row in set, 3 warnings (0.00 sec)
```
`SHOW WARNINGS` statement right after `EXPLAIN` provides detailed information regarding this collation conversion.
```sql
mysql> SHOW WARNINGS;
+---------+------+-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| Level   | Code | Message                                                                                                                                                                                                                                                                             |
+---------+------+-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| Warning | 1739 | Cannot use ref access on index 'contract_id' due to type or collation conversion on field 'contract_id'                                                                                                                                                                             |
| Warning | 1739 | Cannot use range access on index 'contract_id' due to type or collation conversion on field 'contract_id'                                                                                                                                                                           |
| Note    | 1003 | /* select#1 */ select `test`.`teacher`.`id` AS `id`,`test`.`teacher`.`name` AS `name`,`test`.`teacher`.`age` AS `age`,`test`.`teacher`.`contract_id` AS `contract_id`,`test`.`teacher`.`join_on` AS `join_on` from `test`.`teacher` where (`test`.`teacher`.`contract_id` = 100400) |
+---------+------+-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
3 rows in set (0.00 sec)
```

If we compare column `contract_id` with a string (the same data type), the optimizer would be able to use the index and gives the result very efficiently: see column `rows` which indicates that the optimizer is able to locate the exact one row that meets the requirement. 
```sql
mysql> EXPLAIN select * from test.teacher where contract_id = '100400';
+----+-------------+---------+------------+------+---------------+-------------+---------+-------+------+----------+-------+
| id | select_type | table   | partitions | type | possible_keys | key         | key_len | ref   | rows | filtered | Extra |
+----+-------------+---------+------------+------+---------------+-------------+---------+-------+------+----------+-------+
|  1 | SIMPLE      | teacher | NULL       | ref  | contract_id   | contract_id | 22      | const |    1 |   100.00 | NULL  |
+----+-------------+---------+------------+------+---------------+-------------+---------+-------+------+----------+-------+
1 row in set, 1 warning (0.00 sec)
```
We won't see similar warning like the above query:
```sql
mysql> SHOW WARNINGS;
+-------+------+---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| Level | Code | Message                                                                                                                                                                                                                                                                               |
+-------+------+---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| Note  | 1003 | /* select#1 */ select `test`.`teacher`.`id` AS `id`,`test`.`teacher`.`name` AS `name`,`test`.`teacher`.`age` AS `age`,`test`.`teacher`.`contract_id` AS `contract_id`,`test`.`teacher`.`join_on` AS `join_on` from `test`.`teacher` where (`test`.`teacher`.`contract_id` = '100400') |
+-------+------+---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
1 row in set (0.00 sec)
```


## Misunderstanding 3: If `key` is not null in `EXPLAIN` statement output, it means one of the index is used and the query is efficient
Yes, if the value of the `key` column is not null in `EXPLAIN` statement output, it means one of the index is used but it doesn't mean an index is FULLY used.
To decide if a query is optimized, we need to analyze other columns like `key_len`. This column indicates the length of the key the optimizer decided to use with which you can tell how many parts of a composite index are used.
For example, suppose we have a table defined as below and the value for `key_len` is 4

```sql
`col1` char(1) NOT NULL,                   -- 1 byte
`col2` char(2) NOT NULL,                   -- 2 bytes
`col3` tinyint(1) NOT NULL DEFAULT,        -- 1 byte
`col4` tinyint(4) NOT NULL DEFAULT,        -- 4 bytes
KEY `Index` (`col1`, `col2`, `col3`, `col4`)
```
Since indexes are used from left to right (leftmost prefix matching principle), only (`col1`, `col2`, `col3`) is used: 1+2+1=4.

In addition, when using `EXPLAIN` to analyze your query's performance, be careful about the sample data you use - it may produce very different query plan result.

_Reference_: Check out the full `EXPLAIN` output format via https://dev.mysql.com/doc/refman/8.0/en/explain-output.html.

## Misunderstanding 4: The order of columns in `WHERE` clause matters in leftmost prefix principle of index
The truth is, it doesn't. Only the order in the index matters.
**TODO**


