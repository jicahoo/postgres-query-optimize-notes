# postgres-query-optimize-notes
Notes of postgres query

## Common optimize measurements.
## Index

## Generate big table
```sql
create table t_random as select s as id, md5(random()::text) as decription from generate_Series(1,5) s; 
```
## Query Plan

## Slow query pattern:

## Process and memory

## Cache mechansim

## Related repo
* https://github.com/jicahoo/db-posgres-sql
