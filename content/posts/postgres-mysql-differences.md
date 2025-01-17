+++
date = '2025-01-13T19:09:23+05:30'
draft = true
title = 'Postgres MySQL Differences'
+++


MySQL vs postgres

1. PK is clustered in MySQL. Heap based table in PG
2. No table bloat in MySQL, better for update heavy workload compared to Postgres.
3. RLS available in PG, not in MySQL
4. on delete set null with composite PKs and FKs.
5. Indexing json columns
6. check constraints on FK columns don't work in MySQL
7. Declarative partitioning does not work with FKs.
8. Partial indexes not supported in MySQL




### Other considerations
- delete one tenant's data
- rebalancing shards
- schema migrations
  - use data_owner_role
  - replace views
- sequence generator