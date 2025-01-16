+++
date = '2025-01-09T15:09:04+05:30'
draft = false
tags = ['MySQL','multi-tenancy','spring-boot']
title = 'Multi-tenancy with MySQL'
+++

In this article, I present an approach to implement a scalable multi-tenancy architecture with MySQL. The example shown here uses Spring Boot, however, the concepts are general and can be implemented in other programming languages / frameworks.

## Introduction

A SaaS provider typically handles multiple customers. Each customer is a **tenant** from the point of view of the provider. The SaaS provider is responsible for ensuring separation between tenants. The fundamental requirement is that data of one tenant should not be accessible to other tenants.

A key aspect of designing the multi-tenancy architecture  for a web service is **tenant scoping**. i.e. How does the application scope its requests to a particular tenant at a time? The application's context (logged in user in case of a API request), helps us identify the currently active tenant.

There are various multi-tenancy architectures which provide different trade-offs in terms of how much separation they offer, cost, scalability and ease of management. We will look at the approaches and discuss their pros and cons.


### 1. Database instance per tenant
In this approach, each tenant gets their own database instance with separate resources (CPU / memory). This provides the highest levels of tenant separation - even a performance degradation in one tenant’s database instance does not impact other tenants.

For applications, tenant scoping is achieved simply by connecting to the current tenant’s database instance.

However, this approach is **impractical in most cases** due to higher infrastructure costs and management overhead (spinning up new database instances for every new customer).
### 2. Schema per tenant
In this approach, we use shared database instance(s) and each tenant gets their own separate schema (or database).

For applications, tenant scoping is as easy as connecting to the current tenant’s schema. Because the database engine resources (CPU / memory) are shared across tenants, it is possible that costly application queries from one tenant could impact other tenants that share the same database instance.

While this scales better than the first approach, we start facing difficulties after a few thousand tenants. Managing migrations for thousands of schema becomes difficult. Also, database engines have practical limits on number of tables that they can efficiently handle.

AWS RDS for MySQL, for instance, recommends maximum of ten thousand tables per database instance.

> When there is performance degradation because of a large number of tables (more than 10 thousand), it is caused by MySQL working with storage files, including opening and closing them. To address this issue, you can increase the size of the table_open_cache and table_definition_cache parameters. However, increasing the values of those parameters might significantly increase the amount of memory MySQL uses, and might even use all of the available memory. For more information, see [How MySQL Opens and Closes Tables](https://dev.MySQL.com/doc/refman/8.0/en/table-cache.html) in the MySQL documentation.
>
> In addition, too many tables can significantly affect MySQL startup time. Both a clean shutdown and restart and a crash recovery can be affected, especially in versions prior to MySQL 8.0.
>
> We recommend having **fewer than 10,000** tables total across all of the databases in a DB instance. For a use case with a large number of tables in a MySQL database, see [One Million Tables in MySQL 8.0](https://www.percona.com/blog/2017/10/01/one-million-tables-MySQL-8-0/).

**Refernce:** [Best practices for Amazon RDS
](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/CHAP_BestPractices.html)

### 3. Shared schema
In this approach, multiple tenants are mapped to a single schema (or database) and so their data are stored in a set of shared tables. This approach scales better for larger number of tenants as we can pack hundreds or thousands of tenants in a single schema. However, it comes with its **own set of challenges**.

Since the tables are shared across tenants, each row should be mapped to its tenant in some way. For the application, tenant scoping is no longer straight-forward as every database query now needs to filter for the current tenant’s rows only.

Shared schema also results in much larger table sizes compared to the first two approaches and poses a challenge of how to make sure the database engine efficiently processes application queries.

## Implementing multi-tenancy with MySQL

We will see how to implement the shared schema approach with MySQL. Once we implement the shared schema approach with a single schema, we can then easily extend it to support multiple schema and database instances. This will allow us to scale to a large number of tenants easily.

{{< callout text="With 2 database instances, each with 100 schema and each schema holding 200 tenant's data, we would be able to support 40,000 tenants." >}}

### Shared schema data model

In each of the tables, every row needs to be mapped with a particular tenant. For this, we will add a `tenant_id` column to all tables and make it the leading column of the (composite) primary key. Similarly, for every unique constraint, foreign key and secondary index, we will include `tenant_id` as the leading column.

```sql
CREATE TABLE department(
  tenant_id SMALLINT NOT NULL, 
  id INT, 
  name VARCHAR(100) NOT NULL, 
  head_id INT, 
  PRIMARY KEY(tenant_id, id), 
  UNIQUE KEY department_name_idx(tenant_id, name)
);

CREATE TABLE user(
  tenant_id SMALLINT NOT NULL, 
  id INT, 
  name VARCHAR(100), 
  email VARCHAR(255), 
  activated boolean DEFAULT false, 
  reporting_manager_id INT, 
  created_time TIMESTAMP(3), 
  PRIMARY KEY(tenant_id, id), 
  UNIQUE KEY user_email_idx(tenant_id, email)
);

ALTER TABLE department ADD CONSTRAINT department_head_fk FOREIGN KEY (tenant_id, head_id) REFERENCES user(tenant_id, id);

ALTER TABLE user ADD CONSTRAINT user_reporting_manager_fk FOREIGN KEY (tenant_id, reporting_manager_id) REFERENCES user(tenant_id, id);
```
This data model ensures that *one tenant's department can never have another tenant's user as its head*.

**Why is `tenant_id`'s type `smallint`?**

In MySQL (InnoDB), all tables are clustered by primary key. i.e. Primary key is a b+ tree which has the [PK value as key and the entire row as the value](https://www.percona.com/blog/tuning-innodb-primary-keys/). Secondary indexes are also b+ trees that have the indexed column's value as the key and the row's PK as value. Increasing the size of the PK will increase the size of the primary index as well as all secondary indexes on the table.

When using composite primary keys, using `tinyint` for `tenant_id` column (1 byte extra), `256` tenants per schema. With `smallint` (2 bytes extra) we will be able to support up to `65K` tenants per schema.


**Why is `tenant_id` the leading column of every primary key?**

In a multi-tenant application, every query will filter on the `tenant_id` column. We want these queries to execute efficiently. For each table, having `tenant_id` as leading column in the primary key makes all the rows of a tenant clustered. i.e. stored in contiguous data pages. Even when a query has no other criteria (other than the `tenant_id` criteria), the full table would not get scanned - only the portion of the table for that particular tenant would get scanned.

**What should be the datatype for `id` column?**

This depends on the maximum number of rows we expect the table to hold per tenant. `int` would be good enough for the largest table (4,294,967,295). Smaller tables can use smaller integer datatypes. It is important to keep **primary keys as small as possible**. 

### Tenant Scoping

How do we ensure that all application queries mandatorily filter on the `tenant_id` column? Postgres (*since version 9.5*) supports [Row Level Security](https://www.postgresql.org/docs/current/ddl-rowsecurity.html). With Postgres RLS,  it is [easy to implement](https://www.crunchydata.com/blog/row-level-security-for-tenants-in-postgres) [shared schema multi-tenancy](https://docs.aws.amazon.com/prescriptive-guidance/latest/saas-multitenant-managed-postgresql/rls.html) by creating a policy for each table which filters rows of a single tenant.

The approach presented in the linked articles makes tenant scoping trivial for the application. However, **MySQL does not support row level security**. We will see how we can implement a similar solution without RLS in MySQL.

#### Updatable views

Although RLS is not available in MySQL, we can implement something similar using [views](https://dev.MySQL.com/doc/refman/8.0/en/views.html).
> MySQL supports views, including updatable views. Views are stored queries that when invoked produce a result set. A view acts as a **virtual table**.

The application must set the current tenant's id as a session variable `currentTenant`.
```sql
MySQL> set @currentTenant=229;
Query OK, 0 rows affected (0.00 sec)

MySQL> select @currentTenant;
+----------------+
| @currentTenant |
+----------------+
|         229    |
+----------------+
1 row in set (0.01 sec)
```
We will create a view for each of the tables which will filter rows based on the value of `currentTenant`.

MySQL does not allow us to use a session variable in the view definition query directly. i.e. The following create view statement does **not** work.
```sql
MySQL> create view v_department as select * from department where tenant_id =  @currentTenant;
ERROR 1351 (HY000): View's SELECT contains a variable or parameter
```
Instead, we will create a function `current_tenant` that returns the `currentTenant` value and use it in the view definition.
```sql
MySQL> CREATE FUNCTION current_tenant() RETURNS int DETERMINISTIC RETURN CAST(@currentTenant as signed int);
Query OK, 0 rows affected (0.02 sec)
```
Note that the function has been marked as **DETERMINISTIC** - i.e. the function returns the same value when invoked multiple times during the execution of a statement. This allows the query optimiser to create an [efficient query execution plan](https://dev.MySQL.com/doc/refman/8.4/en/function-optimization.html) by invoking the function once and caching the returned value for the entire query plan.


MySQL allows using a function call in the view definition, so the following create view statement works fine.
```sql
MySQL> create view v_department as select * from department where tenant_id =  current_tenant() with check option;
Query OK, 0 rows affected (0.04 sec)
```
The view created here references only a single table and does not have any aggregate function in its select query, hence it is an **updatable view**. i.e. We can execute DML(insert, update, delete) statements on the view and they will automatically be propagated to the underlying table. 

Also note the `with check option` clause. This ensures that insert/update statements cannot insert/update rows that will violate the view definition criteria. i.e. `tenant_id` cannot be set to any other value other than `currentTenant` value.

#### Restricting access to base tables

With the updatable views, when application accesses data via the views, the tenant scoping is done automatically based on the `currentTenant` session variable. But, what if the application directly accesses the underlying table? In that case, we run the risk of application reading/writing data of all tenants.

To solve this, we create two MySQL roles `data_owner` and `app_rw`. We create  two separate schemas - `data` and `app`. Tables are created in `data`  schema and views in `app` schema. We grant access to the `data` schema to `data_owner` role and `app` schema to `app_rw` role.

Applications use the `app_rw` role and so do not have access to the base tables directly. The **only way for applications to access data is via the views which have tenant scoping built-in**.

```sql
CREATE ROLE data_owner;
CREATE USER data_owner_user IDENTIFIED BY 'password';

GRANT data_owner TO 'data_owner_user'@'%';
SET PASSWORD FOR 'data_owner_user'@'%' = 'data_owner_user';

SET DEFAULT ROLE 'data_owner' to 'data_owner_user';


CREATE ROLE app_rw;
CREATE USER app_rw_user IDENTIFIED BY 'password';
grant app_rw to 'app_rw_user'@'%';
SET PASSWORD FOR 'app_rw_user' = 'app_rw_user';
SET DEFAULT ROLE 'app_rw' to 'app_rw_user';

CREATE DATABASE data;
GRANT ALL PRIVILEGES ON data.* to data_owner;

CREATE DATABASE app;
GRANT ALL PRIVILEGES ON app.* to app_rw;

FLUSH PRIVILEGES;
```

Create all the tables and `current_tenant` function in `data` schema.

```sql
CREATE TABLE data.department(
  tenant_id SMALLINT NOT NULL, 
  id INT, 
  name VARCHAR(100) NOT NULL, 
  head_id INT, 
  PRIMARY KEY(tenant_id, id), 
  UNIQUE KEY department_name_idx(tenant_id, name)
);

CREATE TABLE data.user(
  tenant_id SMALLINT NOT NULL, 
  id INT, 
  name VARCHAR(100), 
  email VARCHAR(255), 
  activated boolean DEFAULT false, 
  reporting_manager_id INT, 
  created_time TIMESTAMP(3), 
  PRIMARY KEY(tenant_id, id), 
  UNIQUE KEY user_email_idx(tenant_id, email)
);

ALTER TABLE data.department ADD CONSTRAINT department_head_fk FOREIGN KEY (tenant_id, head_id) REFERENCES user(tenant_id, id);

ALTER TABLE data.user ADD CONSTRAINT user_reporting_manager_fk FOREIGN KEY (tenant_id, reporting_manager_id) REFERENCES user(tenant_id, id);

CREATE FUNCTION data.current_tenant() RETURNS int DETERMINISTIC RETURN CAST(@currentTenant as signed int);

```
Optionally, we could also set up before insert triggers on the base tables so that, the `tenant_id` is automatically populated. This way, the application does not have to do anything related to multi-tenancy and everything is handled in MySQL itself.
```sql
CREATE TRIGGER data.department_tenant BEFORE INSERT ON Department FOR EACH ROW SET new.tenant_id = data.current_tenant();

CREATE TRIGGER data.user_tenant BEFORE INSERT ON User FOR EACH ROW SET new.tenant_id = data.current_tenant();
```
### Example application using spring boot
### Advantages
- application is free to use any technology -- jooq, jpa, native query.
- very few code changes for apps that are already using separate schema / database instance approach to be made into shared schema approach.

### Limitations
- on delete set null
  - simulate using triggers
- cannot use index hints
  - should ideally not use index hints anyways

### Performance
- query plan uses index for FK
- query plan uses index for sorting
- query plan does not scan whole table even when there are no useful indexes.

### Other considerations
- delete one tenant's data
- rebalancing shards
- schema migrations
  - use data_owner_role
  - replace views
- sequence generator