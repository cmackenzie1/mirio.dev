---
title: "Don't use LIMIT OFFSET for pagination in PostgreSQL"
date: "2024-08-03T12:00:00Z"
draft: false
slug: "pagination-in-postgresql"
keywords: [postgresql, pagination, limit, offset, sql, cursor]
tags: [postgresql, sql, pagination]
---

## Introduction

`LIMIT` / `OFFSET` pagination is one of the most common ways to implement pagination in your service. You can find countless tutorials online describing the method and its even the default in the popular web framework Django[^2]. While it gets the job done for most use cases and smaller tables, you may begin to run into performance issues when paginating tables with millions of rows. Django and Laravel[^3] even have warnings in their documentation about this type of pagination.

In the rest of the post I'll show you how `LIMIT` / `OFFSET` pagination performs on larger tables as well as providing an alternative to `LIMIT` / `OFFSET` called cursor-based pagination.

## Setting up

All examples in this post will use the IMDB dataset[^1]. If you wish to follow along and try some queries out yourself, download and import the IMDB dataset into a PostgreSQL databse. If you haven’t installed PostgreSQL, get it from the [official website](https://www.postgresql.org/download/).

Here’s how I set it up using Ubuntu 24.04:

```bash
# Download the IMDB dataset
wget https://dataverse.harvard.edu/api/access/datafile/:persistentId?persistentId=doi:10.7910/DVN/2QYZBT/TGYUNU -O imdb.pgdump
# Create a database
sudo -u postgres createdb imdb
# Import the dataset
sudo -u postgres pg_restore --verbose --clean --no-acl --no-owner --dbname=imdb  -U postgres imdb.pgdump
```

After the import is complete, you can connect to the database using the `psql` command.

```bash
sudo -u postgres psql -U postgres imdb
```

Your database should look something like this:

```text
imdb=# \d
              List of relations
 Schema |      Name       | Type  |  Owner
--------+-----------------+-------+----------
 public | aka_name        | table | postgres
 public | aka_title       | table | postgres
 public | cast_info       | table | postgres
 public | char_name       | table | postgres
 public | comp_cast_type  | table | postgres
 public | company_name    | table | postgres
 public | company_type    | table | postgres
 public | complete_cast   | table | postgres
 public | info_type       | table | postgres
 public | keyword         | table | postgres
 public | kind_type       | table | postgres
 public | link_type       | table | postgres
 public | movie_companies | table | postgres
 public | movie_info      | table | postgres
 public | movie_info_idx  | table | postgres
 public | movie_keyword   | table | postgres
 public | movie_link      | table | postgres
 public | name            | table | postgres
 public | person_info     | table | postgres
 public | role_type       | table | postgres
 public | title           | table | postgres
(21 rows)
```

## Methodology

We’ll use `EXPLAIN ANALYZE` to compare the performance of `LIMIT` and `OFFSET` with other pagination methods. This command provides detailed query execution information, particularly the planning and execution times. We will use the query planning and execution times to compare the performance of the two methods. In each example, we will start at the beginning and then increase the offeset by factors of ten until we reach one million.

This data below is not meant to measure the exact timings of these queries, and the hardware it is running on doesn't really matter here. What does matter is the understanding of how it performs over datasets that vary by orders of magnitude. Additionally, to avoid much bias from query buffers in PostgreSQL, I will "warm" each query by executing it a few times before recording the timings.

## The LIMIT and OFFSET clauses

`LIMIT` and `OFFSET` is often the first choice for pagination because it’s easy to understand and implement. You will often see URLs that look something like: `https://example.com/titles?limit=100&offset=20`, where the `limit` and `offset` parameters are used to construct a matching SQL query:

```sql
SELECT * FROM title ORDER BY id LIMIT 100 OFFSET 20;
```

Each time you click the "Next" button, the `offset` value gets increased by the `limit` value and a new page is returned.

Here’s an example using the `title` table from the IMDB dataset, which contains over 2.5 million rows. Starting on the first page, and then increasing the offsets to `100`, `,1000`, `10,000`, `100,000` and finally `1,000,000`.

```sql
imdb=# EXPLAIN ANALYZE SELECT * FROM title ORDER BY id LIMIT 100 OFFSET 0;
                                                              QUERY PLAN
--------------------------------------------------------------------------------------------------------------------------------------
 Limit  (cost=0.43..4.47 rows=100 width=95) (actual time=0.042..0.233 rows=100 loops=1)
   ->  Index Scan using title_pkey on title  (cost=0.43..102138.95 rows=2528532 width=95) (actual time=0.040..0.209 rows=100 loops=1)
 Planning Time: 0.162 ms
 Execution Time: 0.275 ms
(4 rows)
```

And then again with an `OFFSET` value of 100,000:

```sql
imdb=# EXPLAIN ANALYZE SELECT * FROM title ORDER BY id LIMIT 100 OFFSET 100000;
                                                                QUERY PLAN
------------------------------------------------------------------------------------------------------------------------------------------
 Limit  (cost=4039.87..4043.91 rows=100 width=95) (actual time=65.184..65.247 rows=100 loops=1)
   ->  Index Scan using title_pkey on title  (cost=0.43..102138.95 rows=2528532 width=95) (actual time=0.020..61.460 rows=100100 loops=1)
 Planning Time: 0.148 ms
 Execution Time: 65.277 ms
(4 rows)
```

| Offset    | Planning Time (ms) | Execution Time (ms) |
| --------- | ------------------ | ------------------- |
| 0         | 0.162              | 0.275               |
| 100       | 0.161              | 0.399               |
| 1,000     | 0.190              | 1.653               |
| 10,000    | 0.160              | 13.976              |
| 100,000   | 0.091              | 63.653              |
| 1,000,000 | 0.102              | 492.193             |

Plotting this data gives us the following chart:

![](/img/pagination-in-postgresql-limit-offset.svg)

As we can see in the chart above, we can quickly see that performance for these kinds of queries can get quickly out of hand and queries that used to take less than a millisecond are now taking close to half a second!

## Cursor-based pagination

Another popular pagination method is cursor-based pagination, which uses a unique, ordered column to paginate results. This method is more efficient than `LIMIT` and `OFFSET` because it doesn’t require scanning all the rows before the desired ones and can be used with indexes to further improve performance.

The URL parameters for cursor pagination can be modeled with a `after_id` parameter and the same `limit` parameter we saw before: `https://example.com/titles?after_id=20&limit=100`

And the SQL query would look like this:

```sql
SELECT * FROM title WHERE id > 20 ORDER BY id LIMIT 100;
```

However, sometimes you may see the cursor information encoded using base64 like so: `https://example.com/titles?cursor=YWZ0ZXJfaWQ9MjAmbGltaXQ9MTAw`, but these are not material differences and still represent the same data.

Executing this query on the `title` table with the `after_id` values of `100`, `,1000`, `10,000`, `100,000` and then finally `1,000,000`, we get the following results:

```sql
imdb=# EXPLAIN ANALYZE SELECT * FROM title WHERE id >= 0 ORDER BY id LIMIT 100;
                                                              QUERY PLAN
--------------------------------------------------------------------------------------------------------------------------------------
 Limit  (cost=0.43..4.72 rows=100 width=95) (actual time=0.019..0.105 rows=100 loops=1)
   ->  Index Scan using title_pkey on title  (cost=0.43..108460.28 rows=2528532 width=95) (actual time=0.018..0.096 rows=100 loops=1)
         Index Cond: (id >= 0)
 Planning Time: 0.187 ms
 Execution Time: 0.125 ms
(5 rows)
```

And for `10,000`:

```sql
imdb=# EXPLAIN ANALYZE SELECT * FROM title WHERE id >= 100000 ORDER BY id LIMIT 100;
                                                              QUERY PLAN
--------------------------------------------------------------------------------------------------------------------------------------
 Limit  (cost=0.43..4.72 rows=100 width=95) (actual time=0.020..0.087 rows=100 loops=1)
   ->  Index Scan using title_pkey on title  (cost=0.43..104099.99 rows=2426313 width=95) (actual time=0.020..0.079 rows=100 loops=1)
         Index Cond: (id >= 100000)
 Planning Time: 0.139 ms
 Execution Time: 0.104 ms
(5 rows)
```

| Cursor    | Planning Time (ms) | Execution Time (ms) |
| --------- | ------------------ | ------------------- |
| 0         | 0.197              | 0.152               |
| 100       | 0.249              | 0.298               |
| 1,000     | 0.252              | 0.245               |
| 10,000    | 0.259              | 0.355               |
| 100,000   | 0.198              | 0.250               |
| 1,000,000 | 0.189              | 0.375               |

![](/img/pagination-in-postgresql-cursor.svg)

Cursor pagination yields a significant performance gain when compared to `LIMIT`/`OFFSET` pagination.

## Summary

We've explored two common pagination methods in PostgreSQL: `LIMIT`/`OFFSET` and cursor-based pagination. While `LIMIT`/`OFFSET` is widely used and easy to implement, our performance tests reveal significant drawbacks when dealing with large datasets.

Using the `title` table in IMDB dataset with over 2.5 million rows, we demonstrated how `LIMIT`/`OFFSET` queries degrade as the offset increases. What started as sub-millisecond queries for early pages ballooned to nearly half a second for later pages - a concerning performance hit for any high-traffic application.

In contrast, cursor-based pagination maintained consistent performance regardless of how far into the dataset we paginated. This method leverages database indexes more effectively, resulting in query times that remained under a millisecond even when accessing the millionth row.

The takeaway is clear: while `LIMIT`/`OFFSET` might suffice for smaller datasets or lower-traffic applications, cursor-based pagination is the superior choice for scalable, high-performance systems. As your data grows, the performance gains of cursor pagination become increasingly valuable.

Remember, choosing the right pagination method early can save you from headaches down the road. Don't let inefficient pagination be the bottleneck in your otherwise well-optimized PostgreSQL database!

## A Closer Look at Query Cost

In our earlier examples, we focused primarily on the execution times of our queries. However, the `EXPLAIN ANALYZE` output provides another crucial piece of information: the `cost` field. Let's revisit our query outputs and examine this aspect more closely.

For our `LIMIT`/`OFFSET` query with an offset of 100,000, we saw:

```sql
imdb=# EXPLAIN ANALYZE SELECT * FROM title ORDER BY id LIMIT 100 OFFSET 100000;
                                                                QUERY PLAN
------------------------------------------------------------------------------------------------------------------------------------------
 Limit  (cost=4039.87..4043.91 rows=100 width=95) (actual time=65.184..65.247 rows=100 loops=1)
   ->  Index Scan using title_pkey on title  (cost=0.43..102138.95 rows=2528532 width=95) (actual time=0.020..61.460 rows=100100 loops=1)
 Planning Time: 0.148 ms
 Execution Time: 65.277 ms
(4 rows)
```

The cost field appears as two numbers separated by two dots. The first number represents the estimated startup cost before the first row is retrieved, while the second number is the estimated total cost. These costs are arbitrary units determined by PostgreSQL's query planner based on various factors like I/O operations and CPU processing time.

In this case, we see a startup cost of 4039.87 and a total cost of 4043.91 for the `LIMIT` operation. This high startup cost reflects the work needed to scan through and discard the first 100,000 rows.

Now, let's compare this to our cursor-based pagination query:

```sql
imdb=# EXPLAIN ANALYZE SELECT * FROM title WHERE id >= 100000 ORDER BY id LIMIT 100;
                                                              QUERY PLAN
--------------------------------------------------------------------------------------------------------------------------------------
 Limit  (cost=0.43..4.72 rows=100 width=95) (actual time=0.020..0.087 rows=100 loops=1)
   ->  Index Scan using title_pkey on title  (cost=0.43..104099.99 rows=2426313 width=95) (actual time=0.020..0.079 rows=100 loops=1)
         Index Cond: (id >= 100000)
 Planning Time: 0.139 ms
 Execution Time: 0.104 ms
(5 rows)
```

Here, we see a dramatically lower startup cost of 0.43 and a total cost of 4.72 for the `LIMIT` operation. This lower cost aligns with the much faster execution time we observed.

The cost field provides valuable insights into how PostgreSQL's query planner evaluates different query strategies. In general, lower costs correlate with faster query execution, though this isn't always a one-to-one relationship. The stark difference in costs between these two pagination methods further underscores the efficiency of cursor-based pagination for large offsets.

It's worth noting that while execution times can vary based on system load and data caching, the cost estimates remain constant for a given query and database state. This makes the cost a reliable metric for comparing query efficiency, especially when you're optimizing queries in a development environment where actual execution times might not reflect production conditions.

[^1]: https://doi.org/10.7910/DVN/2QYZBT

[^2]: https://docs.djangoproject.com/en/5.0/ref/paginator/#django.core.paginator.Paginator

[^3]: https://laravel.com/docs/11.x/pagination
