---
layout: post
title:  "Postgres' lateral join, have you heard about it?"
author: "Amanda Sposito"
date:   2020-12-29 09:30:00 -0300
categories:
  - PostgreSQL
  - performance
---

Since [PostgreSQL 9.3](https://www.postgresql.org/docs/9.3/release-9-3.html) we have a new [LATERAL](https://www.postgresql.org/docs/current/queries-table-expressions.html#QUERIES-LATERAL) option for FROM-clause subqueries and function calls.

![](/assets/images/postgres-lateral-join/k8-H1o9rL0XBx0-unsplash.jpg)
Photo by [K8](https://unsplash.com/@k8_iv?utm_source=unsplash&amp;utm_medium=referral&amp;utm_content=creditCopyText) on [Unsplash](https://unsplash.com/?utm_source=unsplash&amp;utm_medium=referral&amp;utm_content=creditCopyText)

### What it is?

According to the documentation what it does is:

> The LATERAL keyword can precede a sub-SELECT FROM item. This allows the sub-SELECT to refer to columns of FROM items that appear before it in the FROM list. (Without LATERAL, each sub-SELECT is evaluated independently and so cannot cross-reference any other FROM item.)

Basically, what it does is that for each row in the main select, it evaluates the sub-select using the main select row as a parameter. Quite similar to a [for loop](https://www.postgresql.org/docs/current/plpgsql-control-structures.html) that iterates through the rows returned by a SQL query.

### TOP N results

Let's think about an example. We need to fetch the most recent items of each tag in a list.

Imagine that we have a table `tags` and a table `movies`, with a one-to-many relationship. To make things easier, a movie can only have one tag. I want to fetch the most recent videos of each tag. How would you do it?

Let's create our tables and populate them with some data, to help us visualize.

```sql
CREATE TABLE tags (
  id serial PRIMARY KEY,
  name VARCHAR(255)
);

CREATE TABLE movies (
  id serial PRIMARY KEY,
  name VARCHAR(255),
  tag_id int NOT NULL,
  created_at timestamp NOT NULL DEFAULT NOW(),
  FOREIGN KEY (tag_id) REFERENCES tags(id) ON UPDATE CASCADE
);

CREATE INDEX movies_tag_id_index ON movies (tag_id);
```

```sql
-- Genres
INSERT INTO "tags"("name") VALUES('Action');
INSERT INTO "tags"("name") VALUES('Animation');
INSERT INTO "tags"("name") VALUES('Sci-Fi');

-- Movies
INSERT INTO "movies"("name", "tag_id", "created_at") VALUES('The Matrix', (SELECT id FROM "tags" where "name" = 'Action'), '1999-05-21');
INSERT INTO "movies"("name", "tag_id", "created_at") VALUES('Tenet', (SELECT id FROM "tags" where "name" = 'Action'), '2020-10-29');
INSERT INTO "movies"("name", "tag_id", "created_at") VALUES('Wonder Woman 1984', (SELECT id FROM "tags" where "name" = 'Action'), '2020-12-25');

INSERT INTO "movies"("name", "tag_id", "created_at") VALUES('Toy Story', (SELECT id FROM "tags" where "name" = 'Animation'), '1995-12-22');
INSERT INTO "movies"("name", "tag_id", "created_at") VALUES('Monsters Inc.', (SELECT id FROM "tags" where "name" = 'Animation'), '2001-11-14');
INSERT INTO "movies"("name", "tag_id", "created_at") VALUES('Finding Nemo', (SELECT id FROM "tags" where "name" = 'Animation'), '2003-07-4');

INSERT INTO "movies"("name", "tag_id", "created_at") VALUES('Arrival', (SELECT id FROM "tags" where "name" = 'Sci-Fi'), '2016-10-24');
INSERT INTO "movies"("name", "tag_id", "created_at") VALUES('Minority Report', (SELECT id FROM "tags" where "name" = 'Sci-Fi'), '2002-08-02');
INSERT INTO "movies"("name", "tag_id", "created_at") VALUES('The Midnight Sky', (SELECT id FROM "tags" where "name" = 'Sci-Fi'), '2020-12-23');
```

### Using row_number to solve it

One way that we can achieve the desired result is to use the `row_number()` [window function](https://www.postgresql.org/docs/current/functions-window.html) combined with an [WITH query (common table expressions)](https://www.postgresql.org/docs/current/queries-with.html).

It will number the rows according to the partition config we specify in the `over` clause, so at the time we fetch the entries we can limit the number of movies from each tag. Let's see how it works.

```sql
SELECT
  tag_id,
  name,
  created_at,
  ROW_NUMBER() OVER(PARTITION BY tag_id ORDER BY tag_id, created_at DESC)
FROM movies;
```

| tag_id | name  | created_at  | row_number |
|---|---|---|---|
| 1 | Wonder Woman 1984 | 2020-12-25 00:00:00 | 1 |
| 1 | Tenet | 2020-10-29 00:00:00 | 2 |
| 1 | The Matrix  | 1999-05-21 00:00:00 | 3 |
| 2 | Finding Nemo  | 2003-07-04 00:00:00 | 1 |
| 2 | Monsters Inc. | 2001-11-14 00:00:00 | 2 |
| 2 | Toy Story | 1995-12-22 00:00:00 | 3 |
| 3 | The Midnight Sky  | 2020-12-23 00:00:00 | 1 |
| 3 | Arrival | 2016-10-24 00:00:00 | 2 |
| 3 | Minority Report | 2002-08-02 00:00:00 | 3 |

We can see that the results are numbered within the tag_id from newer to older. Now we can use it with a CTE and include a condition to only fetch the top 2 of each tag.

```sql
with movies_by_tags (tag_id, name, created_at, rank) as (
  SELECT
    tag_id,
    name,
    created_at,
    ROW_NUMBER() OVER(PARTITION BY tag_id ORDER BY tag_id, created_at DESC)
  FROM movies
)
select *
from movies_by_tags mbt
where mbt.rank < 3
;
```

| tag_id | name  | created_at  | rank |
|---|---|---|---|
| 1 | Wonder Woman 1984 | 2020-12-25 00:00:00 | 1 |
| 1 | Tenet | 2020-10-29 00:00:00 | 2 |
| 2 | Finding Nemo  | 2003-07-04 00:00:00 | 1 |
| 2 | Monsters Inc. | 2001-11-14 00:00:00 | 2 |
| 3 | The Midnight Sky  | 2020-12-23 00:00:00 | 1 |
| 3 | Arrival | 2016-10-24 00:00:00 | 2 |

### Using the lateral keyword to solve it

Now that we understand what we need to do better. A much more simple way to solve this problem would be using the `lateral` keyword.

```sql
SELECT *
FROM tags t
JOIN LATERAL (
  SELECT m.*
  FROM movies m
  WHERE m.tag_id = t.id
  ORDER BY m.created_at DESC
  FETCH FIRST 2 ROWS ONLY
) e1 ON true
;
```

What’s happening is that because of the lateral keyword, we can reference the tags table in the sub-select, and fetch only the movies from that tag, already limiting the results on that step. Without it, we wouldn’t be able to run that query.

```sql
# SELECT *
# FROM tags t
# JOIN (
#   SELECT m.*
#   FROM movies m
#   WHERE m.tag_id = t.id
#   ORDER BY m.created_at DESC
#   FETCH FIRST 2 ROWS ONLY
# ) e1 ON true
# ;
ERROR:  invalid reference to FROM-clause entry for table "t"
LINE 6:   WHERE m.tag_id = t.id
                           ^
HINT:  There is an entry for table "t", but it cannot be referenced from this part of the query.
```

### Comparing the performance of both solutions

Let's run an `explain analyze` and talk about both solutions.

##### Using the row_number

```sql
explain analyze
with movies_by_tags (tag_id, name, created_at, rank) as (
  SELECT
    tag_id,
    name,
    created_at,
    ROW_NUMBER() OVER(PARTITION BY tag_id ORDER BY tag_id, created_at DESC)
  FROM movies
)
select *
from movies_by_tags mbt
where mbt.rank < 3
;
```

```
Subquery Scan on mbt  (cost=16.39..21.29 rows=47 width=536) (actual time=0.066..0.080 rows=6 loops=1)
  Filter: (mbt.rank < 3)
  Rows Removed by Filter: 3
  ->  WindowAgg  (cost=16.39..19.54 rows=140 width=536) (actual time=0.064..0.076 rows=9 loops=1)
        ->  Sort  (cost=16.39..16.74 rows=140 width=528) (actual time=0.048..0.049 rows=9 loops=1)
              Sort Key: movies.tag_id, movies.created_at DESC
              Sort Method: quicksort  Memory: 25kB
              ->  Seq Scan on movies  (cost=0.00..11.40 rows=140 width=528) (actual time=0.012..0.015 rows=9 loops=1)
Planning Time: 0.203 ms
Execution Time: 0.165 ms
```

##### Using the lateral join

```sql
explain analyze
SELECT *
FROM tags t
JOIN LATERAL (
  SELECT m.*
  FROM movies m
  WHERE m.tag_id = t.id
  ORDER BY m.created_at DESC
  FETCH FIRST 2 ROWS ONLY
) e1 ON true
;
```

```
Nested Loop  (cost=8.17..1159.05 rows=140 width=1052) (actual time=0.026..0.040 rows=6 loops=1)
  ->  Seq Scan on tags t  (cost=0.00..11.40 rows=140 width=520) (actual time=0.006..0.007 rows=3 loops=1)
  ->  Limit  (cost=8.17..8.18 rows=1 width=532) (actual time=0.008..0.009 rows=2 loops=3)
        ->  Sort  (cost=8.17..8.18 rows=1 width=532) (actual time=0.007..0.008 rows=2 loops=3)
              Sort Key: m.created_at DESC
              Sort Method: quicksort  Memory: 25kB
              ->  Index Scan using movies_tag_id_index on movies m  (cost=0.14..8.16 rows=1 width=532) (actual time=0.003..0.004 rows=3 loops=3)
                    Index Cond: (tag_id = t.id)
Planning Time: 0.161 ms
Execution Time: 0.068 ms
```

We can see that solutions have similar results, with a few milliseconds of difference between them, and we can also see that the `LATERAL` join performed better.

What happens when we populate the database with a few more of entries? Let's try.

```sql
-- Generates 3_000_000 movies
INSERT INTO "movies"("name", "tag_id")
SELECT
   generate_series(1,1000000) as "name",
   (SELECT id FROM "tags" where "name" = 'Action')
;

INSERT INTO "movies"("name", "tag_id")
SELECT
   generate_series(1,1000000) as "name",
   (SELECT id FROM "tags" where "name" = 'Animation')
;

INSERT INTO "movies"("name", "tag_id")
SELECT
   generate_series(1,1000000) as "name",
   (SELECT id FROM "tags" where "name" = 'Sci-Fi')
;
```

```sql
# select count(*) from movies;
-[ RECORD 1 ]----
count | 150000009
```

Now that we have a lot of data, let's run the `explain analyze` again and see how it goes.

##### Using the row_number

```
Subquery Scan on mbt  (cost=172089.82..181453.23 rows=89175 width=536) (actual time=2207.043..8589.191 rows=6 loops=1)
  Filter: (mbt.rank < 3)
  Rows Removed by Filter: 3000003
  ->  WindowAgg  (cost=172089.82..178109.15 rows=267526 width=536) (actual time=2207.041..8318.158 rows=3000009 loops=1)
        ->  Sort  (cost=172089.82..172758.63 rows=267526 width=528) (actual time=1619.227..2092.200 rows=3000009 loops=1)
              Sort Key: movies.tag_id, movies.created_at DESC
              Sort Method: external merge  Disk: 96624kB
              ->  Seq Scan on movies  (cost=0.00..21784.26 rows=267526 width=528) (actual time=0.038..530.664 rows=3000009 loops=1)
Planning Time: 0.127 ms
Execution Time: 8595.917 ms
```

Let's break it down.

Taking a closer look into the output, we can see that the `WindowAgg` and the `Subquery Scan` on the CTE are the two most expensive actions in the query.

First we do a `sequential scan` on the movies table. This means that we scan through every page of data sequentially, read the data, apply the filter (if exists) and then discard the rows that don't fit. This kind of scan **always** reads everything in the table. Unless we need a large portion of the table, this is generally inefficient.

Then we apply the `sort`, and we see that we are using the `external merge` instead of the `quicksort` from the first `explain analyze`. This happens when the data does not fit in memory, and we use the disk to process it. This is probably the slowest sort method since there is a lot of back and forth to the disk.

After all that, we read from the CTE and apply our filter (mbt.rank < 3) that removes 3_000_003 rows.

##### Using the lateral join

```
Nested Loop  (cost=44189.51..6186549.45 rows=280 width=542) (actual time=315.465..928.772 rows=6 loops=1)
  ->  Seq Scan on tags t  (cost=0.00..11.40 rows=140 width=520) (actual time=0.011..0.014 rows=3 loops=1)
  ->  Limit  (cost=44189.51..44189.52 rows=2 width=22) (actual time=309.581..309.581 rows=2 loops=3)
        ->  Sort  (cost=44189.51..46689.52 rows=1000003 width=22) (actual time=309.578..309.578 rows=2 loops=3)
              Sort Key: m.created_at DESC
              Sort Method: top-N heapsort  Memory: 25kB
              ->  Index Scan using movies_tag_id_index on movies m  (cost=0.43..34189.48 rows=1000003 width=22) (actual time=0.033..193.517 rows=1000003 loops=3)
                    Index Cond: (tag_id = t.id)
Planning Time: 0.165 ms
Execution Time: 928.809 ms
```

Let's break it down.

Starting form the bottom, we can see that we are doing a `index scan` using the index that we created when we created the tables. This means that we can read only some of the rows in a table, and since we want a small set of rows, postgres can directly ask the index. This is faster than the seq scan.

Then we apply the `sort`, and we see that we are using the `top-N heapsort` in this case. This sort is used when you only want a couple of sorted rows. Which fits perfectly for our case of fetching the 2 most recent movies.

After that, we limit and perform the join using a nested loop (where the right relation is scanned once for every row found in the left relation), and then return.

### Conclusion

We can see from the output that the execution time using the `row_number` was around **8.5s**, and the `lateral` was around **0.9s**.

The `row_number` solution has to do a lot more work, reading through all the table rows and remove a lot of rows, so it can return the desired output. This is visible by its execution time. While the `lateral` join solution is able to make use of indexes and a better sort method.

There is probably a few things we could do to optimize both the `row_number` solution and the `lateral` one. But we in general, for this TOP-N case, the `lateral join` is a better fit.

I hope that helps!
