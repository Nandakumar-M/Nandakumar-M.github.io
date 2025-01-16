+++
date = '2025-01-13T19:09:23+05:30'
draft = true
title = 'Postgres MySQL Differences'
+++


MySQL vs postgres

1. PK is clustered in MySQL. Heap based table in PG
2. RLS available in PG, not in MySQL
3. on delete set null with composite PKs and FKs.
4. Indexing json columns
5. check constraints on FK columns don't work on MySQL
