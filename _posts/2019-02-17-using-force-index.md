---
layout: post
title: "Using FORCE INDEX: When Your Query Optimizer Gets It Wrong"
date:   2019-02-17 09:00:00 -0800
categories: mysql database
published: true
---

Unlike most code a developer writes, writing SQL only requires us to describe what data we want and not how to get it. When given a query like `SELECT id, author_id FROM posts WHERE author_id = 123 ORDER BY id`, you needn’t concern yourself with what indexes are used (if any), what type of sort is used, or any other number of implementation details. Instead, the query optimizer handles this for you. This keeps SQL concise and readable and, for the most part, the query optimizer chooses the best path for a given set of data.

But sometimes the query optimizer gets it wrong. Let’s look at one example of this and what to do about it.

We have a blog app that pulls from a posts table that looks like this:

```sql
CREATE TABLE `posts` (
 `id` bigint(20) NOT NULL AUTO_INCREMENT,
 `body` text,
 `author_id` bigint(20) NOT NULL,
 `coauthor_id` bigint(20) DEFAULT NULL,
PRIMARY KEY (`id`),
 KEY `index_posts_on_author_id` (`author_id`),
 KEY `index_posts_on_coauthor_id` (`coauthor_id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8
```

Our app is doing really great and there are over 100 million rows in the posts table. Unfortunately, the coauthor feature wasn’t as popular as expected and is rarely used:

```
mysql> SELECT COUNT(*) FROM posts WHERE coauthor_id IS NOT NULL;
+----------+
| COUNT(*) |
+----------+
| 159286 |
+----------+
1 row in set (0.04 sec)
```

Only about 0.01% of the posts have a coauthor, leaving 99.9% of the rows NULL. When MySQL calculates index statistics, the coauthor_id index comes back with extremely low cardinality: only 29!

```
mysql> show indexes from posts;
+-------+------------+----------------------------+--------------+-------------
+-----------+-------------+----------+--------+------+------------+---------
+---------------+
| Table | Non_unique | Key_name                   | Seq_in_index | Column_name
| Collation | Cardinality | Sub_part | Packed | Null | Index_type | Comment |
Index_comment |
+-------+------------+----------------------------+--------------+-------------
+-----------+-------------+----------+--------+------+------------+---------
+---------------+
| posts |          0 | PRIMARY                   |         1 | id
| A        | 112787604 |        NULL | NULL  |     | BTREE     |        |
|
| posts |          1 | index_posts_on_author_id  |         1 | author_id
| A        | 2891989 |          NULL | NULL |      | BTREE     |        |
|
| posts |          1 | index_posts_on_coauthor_id |        1 | coauthor_id
| A        | 29 |               NULL | NULL |  YES | BTREE     |        |
|+-------+------------+----------------------------+--------------+-------------
+-----------+-------------+----------+--------+------+------------+---------
+---------------+ 3 rows in set (0.00 sec)
```

When an index has such low cardinality, MySQL is less likely to use it. After all, what’s the point of an index that can eliminate very few values?

However, even with such low cardinality, the query optimizer chooses to use it when looking up posts by a coauthor:

```
mysql> EXPLAIN SELECT * FROM posts WHERE coauthor_id = 98543\G
*************************** 1. row ***************************
           id: 1
  select_type: SIMPLE
        table: posts
         type: ref
possible_keys: index_posts_on_coauthor_id
          key: index_posts_on_coauthor_id
      key_len: 9
          ref: const
         rows: 2
        Extra: NULL
1 row in set (0.00 sec)

mysql> SELECT * FROM posts WHERE coauthor_id = 98543;
+----------+--------------+-----------+-------------+
| id       | body         | author_id | coauthor_id |
+----------+--------------+-----------+-------------+
| 21168595 | Lipsum Lorem |         1 |       98543 |
| 25695860 | Lipsum Lorem |     99833 |       98543 |
+----------+--------------+-----------+-------------+
2 rows in set (0.01 sec)
```

This even works when looking up multiple coauthors in a single query:

```
mysql> EXPLAIN SELECT * FROM posts WHERE coauthor_id IN
(98543,99592,99096,98022,99643,99091,98578,99910,99910,98842)\G
*************************** 1. row ***************************
           id: 1
  select_type: SIMPLE
        table: posts
         type: range
possible_keys: index_posts_on_coauthor_id
          key: index_posts_on_coauthor_id
      key_len: 9
          ref: NULL
         rows: 19
        Extra: Using index condition
1 row in set (0.00 sec)

mysql> SELECT * FROM posts WHERE coauthor_id IN
(98543,99592,99096,98022,99643,99091,98578,99910,99910,98842);
+-----------+--------------+-----------+-------------+
| id | body | author_id | coauthor_id |
+-----------+--------------+-----------+-------------+
|  <results removed for brevity>                      |
+-----------+--------------+-----------+-------------+
19 rows in set (0.01 sec)
```

But things go off the rails when we add an extra ID to the IN clause:

```
mysql> EXPLAIN SELECT * FROM posts WHERE coauthor_id IN
(98543,99592,99096,98022,99643,99091,98578,99910,99910,98842,98511)\G
*************************** 1. row ***************************
           id: 1
  select_type: SIMPLE
        table: posts
         type: ALL
possible_keys: index_posts_on_coauthor_id
          key: NULL
      key_len: NULL
          ref: NULL
         rows: 112787604
        Extra: Using where
1 row in set (0.01 sec)

mysql> SELECT * FROM posts WHERE coauthor_id IN
(98543,99592,99096,98022,99643,99091,98578,99910,99910,98842,98511);
+-----------+--------------+-----------+-------------+
| id        | body         | author_id | coauthor_id |
+-----------+--------------+-----------+-------------+
| <results removed for brevity>                      |
+-----------+--------------+-----------+-------------+
21 rows in set (37.39 sec)
```

Instead of using an index, the query optimizer has decided to scan the entire table. What happened?

To understand why this happens, we need to look at how MySQL picks an index. Generally, the best index is the one that matches the fewest rows. To estimate how many rows will be found in each index for a given query, the query optimizer can use one of two methods: using “dives” into the index or using index statistics.

Index dives — counting entries in an index for a given value — provides accurate row count estimates because an index dive is done for each index and value (e.g., for an IN with 10 values, it performs 10 index dives). This can degrade performance. Using index statistics avoids this problem, providing near constant time lookups by reading the table’s index statistics (i.e., cardinality). (More detail on how this is done can be found here.)

In other words: slow and accurate or fast and sloppy. For tables with evenly distributed data, the latter works great. However, when an index is sparsely populated, like with coauthor_id, index statistics can be wildly inaccurate.

In cases like this, we can lend a hand using index hints like USE INDEX and FORCE INDEX. Let’s start with USE INDEX , which according to the documentation, “tells MySQL to use only one of the named indexes to find rows in the table”:

```
mysql> EXPLAIN SELECT * FROM posts USE INDEX(index_posts_on_coauthor_id) WHERE
coauthor_id IN
(98543,99592,99096,98022,99643,99091,98578,99910,99910,98842,98511)\G
*************************** 1. row ***************************
           id: 1
  select_type: SIMPLE
        table: posts
         type: ALL
possible_keys: index_posts_on_coauthor_id
          key: NULL
      key_len: NULL
          ref: NULL
         rows: 112787604
        Extra: Using where
1 row in set (0.00 sec)
This recommendation wasn't enough; MySQL still decided it was better to use no index at all. Let’s try again but this time use FORCE INDEX, which “acts like USE INDEX, with the addition that a table scan is assumed to be very expensive”:

mysql> EXPLAIN SELECT * FROM posts FORCE INDEX(index_posts_on_coauthor_id)
WHERE coauthor_id IN
(98543,99592,99096,98022,99643,99091,98578,99910,99910,98842,98511)\G
*************************** 1. row ***************************
           id: 1
  select_type: SIMPLE
        table: posts
         type: range
possible_keys: index_posts_on_coauthor_id
          key: index_posts_on_coauthor_id
      key_len: 9
          ref: NULL
         rows: 37761660
        Extra: Using index condition
1 row in set (0.00 sec)

mysql> SELECT * FROM posts FORCE INDEX(index_posts_on_coauthor_id) WHERE
coauthor_id IN
(98543,99592,99096,98022,99643,99091,98578,99910,99910,98842,98511);
+-----------+--------------+-----------+-------------+
| id        | body         | author_id | coauthor_id |
+-----------+--------------+-----------+-------------+
| <results removed for brevity>                      |
+-----------+--------------+-----------+-------------+
21 rows in set (0.00 sec)
```

Thankfully, this does the trick, coercing the optimizer into a query plan we know to be more performant. Instead of 37 seconds, the query finishes in less than 1ms!

But why did the behavior change when adding one more ID to the query above, increasing it from 10 to 11 values? In MySQL 5.6, a new variable called eq_range_index_dive_limit was added, setting a threshold for equality ranges like IN.

```
mysql> SHOW VARIABLES LIKE 'eq_range_index_dive_limit';
+---------------------------+-------+
| Variable_name             | Value |
+---------------------------+-------+
| eq_range_index_dive_limit | 10    |
+---------------------------+-------+
1 row in set (0.00 sec)
```

By default, it is set to 10 in MySQL 5.6 and was increased to 200 in MySQL 5.7.4.

While intended to improve performance by reducing index dives, this variable surprised some teams − including my own − by significantly degrading performance of certain queries. When a routinely run query degrades in performance by 10,000x overnight, your database quickly becomes occupied entirely by those queries, grinding your app and your business to a halt. There’s a detailed look at one example of this from the MySQL at Facebook blog.

The query optimizer has been tuned to handle the most common data shapes and, in most cases, will choose a good query plan. However, when this is not the true, FORCE INDEX provides a way to influence the choices of the query optimizer. However, this heavy-handed approach comes with added responsibility: you’re now responsible to specify what data you want and how MySQL should retrieve it. As data evolves and new queries are introduced, the index you’ve forced MySQL to use may no longer be best.

It’s worth considering why the optimizer chooses a catastrophic query plan. In our example, based on a real world system, the shape of our data poorly fit the schema we’d chosen. Instead of continuing to optimize the query using FORCE INDEX, it may be time to roll up our sleeves and change the schema.
