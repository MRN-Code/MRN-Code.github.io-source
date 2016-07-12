authorId: dlandis
title: SQL Lateral Joins
date: 2016-07-12 10:44:40
tags:
---

Recently, I heard about a relatively new (as of 9.3) feature in Postgres called a LATERAL JOIN.  A LATERAL JOIN enables a subquery in the _from_ part of a clause to reference columns from preceding items in the _from_ list (quoted from [here](https://wiki.postgresql.org/wiki/What%27s_new_in_PostgreSQL_9.3#LATERAL_JOIN)).  Without LATERAL, each subquery is evaluated independently and so cannot cross-reference any other _from_ item (quoted from [here](https://www.postgresql.org/docs/9.4/static/queries-table-expressions.html)).

I’ve been looking for a good way to use this feature in our internal code and I finally found one.  In our specific instance, using a LATERAL JOIN sped up a query by an order of magnitude!  However, our example was relatively complex and specific to us, so here is a generic (very contrived) example.

## Setup:
```
-- Create a table with 10 million rows of 6 character random strings of numbers
-- takes about 30 sec to create
CREATE TABLE rand_table AS
   SELECT
      id,
      LPAD(FLOOR(RANDOM()*1000000)::text, 6, '0') AS rand_string
   FROM GENERATE_SERIES(1,10000000) id;

-- Create table with 999,999 rows of 6 character strings of numbers (from 1 to 999,999)
-- takes about 1 sec to create
CREATE TABLE series_table AS
   SELECT
      id,
      LPAD(id::text, 6, '0') AS series
   FROM GENERATE_SERIES(1,999999) id

-- Vacuum analyze both tables
VACUUM ANALYZE rand_table;
VACUUM ANALYZE series_table;
```

## Test
Let's count how many instances of `'010170'` there are in `rand_table`.  Then we'll LEFT JOIN that to the `series_table`.  Like, I said, super contrived...
```
SELECT
   st.id AS series_id,
   st.series,
   rt.count
FROM series_table st

-- Fast query (using a LATERAL JOIN) ~2 seconds
/*
LEFT JOIN LATERAL (
   SELECT
      rand_string,
      COUNT(rand_string)
   FROM rand_table
   WHERE rand_string = st.series -- this is the lateral magic!
   GROUP BY rand_string
) rt ON st.series = rt.rand_string
*/

-- Slow query (using a regular JOIN) ~10 seconds
/*
LEFT JOIN (
   SELECT
      rand_string,
      COUNT(rand_string)
   FROM rand_table
   GROUP BY rand_string
) rt ON st.series = rt.rand_string
*/
WHERE st.id = 10170
```

## Clean Up
```
DROP TABLE rand_table;
DROP TABLE series_table;
```

In the slow (LEFT JOIN) query, the subquery is forced to return _all_ data, and then JOIN on that entire result set.  It takes a long time to grab and count the entire result set of the subquery, which considerably increases the overall query time.  In the fast (LEFT JOIN LATERAL) query, the subquery is able to reference the st.series column from a preceding item, and pare down the subquery result set to only include data that will ultimately be JOINed upon.

As stated, this example is super contrived and there are definitely other ways to rewrite and improve it, but hopefully this will give you the gist of how a LATERAL JOIN should look and function.  Also note, this query speed only improved by about 5 times, however for our internal query, we were able to improve query time by an entire order of magnitude.  Depending where you use LATERAL JOINs, some queries will improve more than others.

Hope that helps!
