# postgres-query-optimize-notes
Notes of postgres query

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

## Slow query pattern:

## Process and memory

## Cache mechansim

## Related repo
* https://github.com/jicahoo/db-posgres-sql
