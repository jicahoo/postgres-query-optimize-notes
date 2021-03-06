# postgres-query-optimize-notes
Notes of postgres query

## Notes of query performance
* Gigantic IN clause is problematic; Replace it with IN VALUES which will create temp table.
* CTE a.k.a WITH clause, it is not temp table, just expression or sub query.
* Hash Join consume memory. May generate many temp files for buckets if there is no enough work memory, which lead to bad performance. Be cautious.
* Avoid join based on function call or expression, unless you can add expression index on it.
* Postgres is good at handling many concurrent queries and each query involve not too much data. 
* Avoid unanchored text query StrColumn like '%foo%'. Or try pg\_trgm index.
* Avoid UNION ALL? UNION vs UNION ALL? Index can't be leveraged?
* TABLE, INDEX, TEMP TABLE.
* Query Plan: InitPlan -- One-time function call and remember the result ($N) to be used in other places.
* Some DB connection middleware will use Prepared statement. The performance of preparted statement and direct statement (sql) are often differrent. 
* 

## In query plan, how to estimate the size of result
* https://www.postgresql.org/docs/current/static/row-estimation-examples.html keywords: histgram, MCV(most common values)

## Common optimize measurements.
## Index

## Generate big table
```sql
-- Way 1:
create table t_random as select s as id, md5(random()::text) as decription from generate_Series(1,5) s; 
-- Way 2:
CREATE TABLE fake_snap (id INTEGER, oneSnap INTEGER, twoSnap INTEGER, description TEXT);
INSERT INTO fake_snap (id, oneSnap, twoSnap, description)
   (SELECT 
        i,
        CASE WHEN i%2 = 0 THEN (i+1)%20000 ELSE NULL END,
        CASE WHEN i%2 != 0 THEN (i+1)%20000 ELSE NULL END,
        md5(random()::text)
    FROM generate_series(0, 20000) as i);
```
## Query Plan

## Slow query pattern and improve tips
### Use expression or function call in JOIN's condition.
* If you do so, query plan can't leverage index. think one example.
### IN vs IN VALUES
* VALUES is a temp table. So query plan can JOIN.

### Multiple self JOINS
* We can leverage CASE WHEN to dedup slow self JOINS.
* Think why nested loop is, sometimes, better than hash join when there is index? 
   * HashJoin need Seq scan both of jonied tables.
   * If there index, for every loop, it can leverage the index of find the data. It won't need to load all data of the joined table.

* Table and data
```SQL
CREATE TABLE fake_snap (id INTEGER, oneSnap INTEGER, twoSnap INTEGER, description TEXT);
INSERT INTO fake_snap (id, oneSnap, twoSnap, description)
   (SELECT 
        i,
        CASE WHEN i%2 = 0 THEN (i+1)%20000 ELSE NULL END,
        CASE WHEN i%2 != 0 THEN (i+1)%20000 ELSE NULL END,
        md5(random()::text)
    FROM generate_series(0, 20000) as i);
```

* 8ms (there is index on id)
```sql
select snap1.id, CASE WHEN snap1.onesnap is not null THEN snap2.id ELSE NULL END AS snap2_id_1, CASE WHEN snap1.onesnap is not null THEN snap2.description ELSE NULL END AS snap2_desc_1, CASE WHEN snap1.twosnap is not null THEN snap2.id ELSE NULL END AS snap2_id_2, CASE WHEN snap1.twosnap is not null THEN snap2.description ELSE NULL END AS snap2_desc_2 from fake_snap snap1 LEFT JOIN fake_snap snap2  ON (snap1.onesnap = snap2.id  or  snap1.twosnap = snap2.id) offset 1000 limit 2000;
```
* 961.875 ms (there is index on id)
```sql
select snap1.id, snap2.id, snap2.description, snap3.id, snap3.description from fake_snap snap1 LEFT JOIN fake_snap snap2  ON snap1.onesnap = snap2.id LEFT JOIN fake_snap snap3 ON snap1.twosnap = snap3.id offset 1000 limit 2000;
```

* 17 seconds (there is index on id). It is slow since we used expression in '=', so the query plan can't leverage index. So avoid it.
```sql
select snap1.id, CASE WHEN snap1.onesnap is not null THEN snap2.id ELSE NULL END AS snap2_id_1, CASE WHEN snap1.onesnap is not null THEN snap2.description ELSE NULL END AS snap2_desc_1, CASE WHEN snap1.twosnap is not null THEN snap2.id ELSE NULL END AS snap2_id_2, CASE WHEN snap1.twosnap is not null THEN snap2.description ELSE NULL END AS snap2_desc_2 from fake_snap snap1 LEFT JOIN fake_snap snap2  ON (snap1.onesnap = snap2.id  or  snap1.twosnap = (snap2.id+1-1)) offset 1000 limit 2000;
```

* 13448.501 ms (there is no index on id) Slow because nested loop O(m*n). 
```sql
select snap1.id, CASE WHEN snap1.onesnap is not null THEN snap2.id ELSE NULL END AS snap2_id_1, CASE WHEN snap1.onesnap is not null THEN snap2.description ELSE NULL END AS snap2_desc_1, CASE WHEN snap1.twosnap is not null THEN snap2.id ELSE NULL END AS snap2_id_2, CASE WHEN snap1.twosnap is not null THEN snap2.description ELSE NULL END AS snap2_desc_2 from fake_snap snap1 LEFT JOIN fake_snap snap2  ON (snap1.onesnap = snap2.id  or  snap1.twosnap = snap2.id) offset 1000 limit 2000;
```
* 788.609 ms ms (there is no index on id)
```sql
select snap1.id, snap2.id, snap2.description, snap3.id, snap3.description from fake_snap snap1 LEFT JOIN fake_snap snap2  ON snap1.onesnap = snap2.id LEFT JOIN fake_snap snap3 ON snap1.twosnap = snap3.id offset 1000 limit 2000;
```

## Process and memory

## Cache mechansim
* https://madusudanan.com/blog/understanding-postgres-caching-in-depth/
* http://fibrevillage.com/database/134-useful-sqls-to-check-contents-of-postgresql-shared-buffer 
## Related repo
* https://github.com/jicahoo/db-posgres-sql

## Reference:
* http://pydanny-event-notes.readthedocs.io/en/latest/DjangoConEurope2012/10-steps-to-better-postgresql-performance.html
