---
layout: post
title:  "Postgres Internal"
date:   2023-09-28 10:32:19 -0400
categories: jekyll update
---
# Postgres Internal

Postgres database cluster is not like many database servers. It hosts on single server and contains multiple databases. Each database contains multiple indexes, tables and other objects. Databases, tables, indexes files are all objects, and are uniquely identified with an **OID**.

![image-20230925110834969](/assets/imgs/storage.png)

Its storage is like a file system. All the files except for tablespace are stored in base directory, specified by `$PGDATA`. Database subdirectories are named as their **OID**s.

Inside database, tables and index files are initially 1GB file named as their **reffilenode**, or serveral 1GB files named as **reffilenode.x** if it gets larger size.

![](/assets/imgs/reffilenode.jpg)

Besides, each table also associates with two files with suffix **_fsm** and **_vm**. They are free space map and visibility map. 

- The **free space map** stores free space of each page within the table file. It is related to **concurrency control**.
- The **visibility map** stores the visibility of each page within the table file. It is related to **Vaccum processing**.

## Table Storage

![](/assets/imgs/heap_file.jpg)

## Indexes

- Hash Index

- Bitmap Index

  - ```
    // for this table
    ----------------
    row_id row_val
    1				25
    2				30
    3				25
    -----------------
    // bitmap index records the distinct values and which rows contain the specific values
    -----------------
    25      1,3
    30      2
    -----------------
    ```

![](/AveryQi115.github.io/assets/imgs/bitmap.png)

- B+Tree Index

References:

- [Indexes in PostgreSQL — 1](https://postgrespro.com/blog/pgsql/3994098)
- [Indexes in PostgreSQL — 2](https://postgrespro.com/blog/pgsql/4161264)
- [Indexes in PostgreSQL — 3 (Hash)](https://postgrespro.com/blog/pgsql/4161321)
- [Indexes in PostgreSQL — 4 (Btree)](https://postgrespro.com/blog/pgsql/4161516)
- [Indexes in PostgreSQL — 5 (GiST)](https://postgrespro.com/blog/pgsql/4175817)
- [Indexes in PostgreSQL — 6 (SP-GiST)](https://habr.com/en/company/postgrespro/blog/446624/)
- [Indexes in PostgreSQL — 7 (GIN)](https://habr.com/en/company/postgrespro/blog/448746/)
- [Indexes in PostgreSQL — 9 (BRIN)](https://habr.com/en/company/postgrespro/blog/452900/)

## Process

![](/assets/imgs/processes.jpg)

Postgres has following main processes:

- **Server process**: parent of all processes. It starts the background processes, replication-associated processes, listens to port 5432 for client connection and starts a backend process for handling that client queries.

- **Backend process**: Each backend process can access only one database. And each is initialized for one client connection. Postgres does not have a connection pool, if the users connect and deconnect frequently the performance is harmed. Usually there are some connection pool middleware to solve the problem.

- **Background process**: 

  ![](/assets/imgs/bg_processes.jpg)

- **Background worker process**: like background process, but is created by user to do background works.

## Memory

![](/assets/imgs/memory.png)

- Local Memory Area: Each backend process allocate a local memory area for query processing
  - **Work-mem**: for sorting tuples for **ORDER_BY** and **DISTINCT**, and for joining tables for **merge join** and **hash join**
  - **maintainence work memory**: for maintenance operations: **VACUUM, REINDEX**
  - **Temp buffer**: store temporary tables
- Shared Memory Area
  - **shared buffer pool**
  - **WAL buffer**: is a buffering area of the WAL data before writing to a persistent storage.
  - **commit log**: keeps the state of all transactions for the concurrency control mechanism

## Query Processing

![](/assets/imgs/query.png)

- Parser

  It is like a frontend of compilers, creating a syntax tree and checking syntax.

  ![](/assets/imgs/parser.png)

- Analyzer

  Generate query tree with target tree, range table, join tree and sort clause.

- Rewriter

  The rewriter is the system that realizes the [rule system](http://www.postgresql.org/docs/current/static/rules.html). For example, views related rules are applied through rewriter.

- Planner

- Executor

### Cost Estimation

Postgres query optimization is based on cost, and costs are indicators to compare relative performance of operations (**not real performance, it is estimated by certain calculations**). Costs have the following types:

- **Start up**

  Start up is the cost expended before the first tuple is fetched. For example, index scan spends start up costs to read index files.

- **Run**

  The cost of fetching all tuples.

- **Total**:Start up + Run

```sql
testdb=# EXPLAIN SELECT * FROM tbl;
                       QUERY PLAN                        
---------------------------------------------------------
 Seq Scan on tbl  (cost=0.00..145.00 rows=10000 width=8)
(1 row)
```

The above query's startup cost is `0.00` and total cost is `145.00`.

**Sequential Scan Estimation**:

![image-20230928111537670](/assets/imgs/seqscan_run_cost.png)

**Index Scan Estimation**:

- **startup cost**

  Although postgres have multiple index, they all use cost_index to estimate. $H_{index}$ is the height of the index tree.

  ![image-20230928112315174](/assets/imgs/index_startup.png)

- **run cost**

  ![image-20230928112834150](/assets/imgs/index_run_cost.png)
