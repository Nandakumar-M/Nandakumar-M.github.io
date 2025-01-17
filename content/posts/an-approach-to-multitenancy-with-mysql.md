+++
date = '2025-01-09T15:09:04+05:30'
draft = false
tags = ['MySQL','multi-tenancy','spring-boot']
title = 'Multi-tenancy with MySQL'
+++

In this article, I present an approach to implement a scalable multi-tenancy architecture with MySQL. The example shown here uses Spring Boot, however, the concepts are general and can be implemented in other programming languages / frameworks.

## Introduction

A SaaS provider typically handles multiple customers. Each customer is a **tenant** from the point of view of the provider. The SaaS provider is responsible for isolating tenant data and operations, ensuring that one tenant's data remains inaccessible to other tenants.

A key aspect of designing the multi-tenancy architecture  for a web service is **tenant scoping**. i.e. How does the application scope its requests to a particular tenant at a time? The application's context (logged in user in case of a API request), helps us identify the currently active tenant.

There are various multi-tenancy architectures which provide different trade-offs in terms of how much separation they offer, cost, scalability and ease of management. We will look at the approaches and discuss their pros and cons.


### 1. Database instance per tenant
In this approach, each tenant gets their own database instance with separate resources (CPU / memory). This provides the highest levels of separation - even a performance degradation in one tenant’s database instance does not impact other tenants.

For applications, tenant scoping is achieved simply by connecting to the current tenant’s database instance.

However, this approach is **impractical in most cases** due to higher infrastructure costs and management overhead (spinning up new database instances for every new customer).
### 2. Schema per tenant
In this approach, we use shared database instance(s) and each tenant gets a separate schema (or database).

For applications, tenant scoping is as easy as connecting to the current tenant’s schema. Because the database engine resources (CPU / memory) are shared across tenants, it is possible that costly application queries from one tenant could impact other tenants that share the same database instance.

While this scales better than the first approach, we start facing difficulties after a few thousand tenants. Managing migrations for thousands of schema becomes difficult. Also, database engines have practical limits on number of tables that they can efficiently handle.

AWS RDS for MySQL, for instance, recommends maximum of ten thousand tables per database instance.

> When there is performance degradation because of a large number of tables (more than 10 thousand), it is caused by MySQL working with storage files, including opening and closing them. To address this issue, you can increase the size of the table_open_cache and table_definition_cache parameters. However, increasing the values of those parameters might significantly increase the amount of memory MySQL uses, and might even use all of the available memory. For more information, see [How MySQL Opens and Closes Tables](https://dev.MySQL.com/doc/refman/8.0/en/table-cache.html) in the MySQL documentation.
>
> In addition, too many tables can significantly affect MySQL startup time. Both a clean shutdown and restart and a crash recovery can be affected, especially in versions prior to MySQL 8.0.
>
> We recommend having **fewer than 10,000** tables total across all of the databases in a DB instance. For a use case with a large number of tables in a MySQL database, see [One Million Tables in MySQL 8.0](https://www.percona.com/blog/2017/10/01/one-million-tables-MySQL-8-0/).

**Reference:** [Best practices for Amazon RDS
](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/CHAP_BestPractices.html)

### 3. Shared schema
In this approach, multiple tenants are mapped to a single schema (or database) and so their data are stored in a set of shared tables. This approach scales better for larger number of tenants as we can pack hundreds or thousands of tenants in a single schema. However, it comes with its own set of challenges.

Since the tables are shared across tenants, each row should be mapped to its tenant in some way. For the application, tenant scoping is no longer straight-forward as every query now needs to filter for the current tenant’s rows only.

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

In MySQL (InnoDB), all tables are clustered by primary key. i.e. Primary key is a b+ tree which has the [PK value as key and the entire row as the value](https://www.percona.com/blog/tuning-innodb-primary-keys/). Secondary indexes are also b+ trees that have the indexed column's value as the key and the row's PK as value.

Increasing the size of the PK will increase the size of the primary index as well as all secondary indexes on the table.

When using composite primary keys, using `tinyint`(1 byte) for `tenant_id` column, 256 tenants per schema can be supported. With `smallint`(2 bytes) we will be able to support up to 65K tenants per schema.


**Why is `tenant_id` the leading column of every primary key?**

In a multi-tenant application, every query will filter on the `tenant_id` column. We want these queries to execute efficiently. For each table, having `tenant_id` as leading column in the primary key makes all the rows of a tenant clustered. i.e. stored in contiguous data pages. Even when a query has no other criteria (other than the `tenant_id` criteria), the full table would not get scanned - only the portion of the table for that particular tenant would get scanned.

**What should be the datatype for `id` column?**

This depends on the maximum number of rows we expect the table to hold per tenant. `int` would be good enough for the largest table (4,294,967,295). Smaller tables can use smaller integer datatypes. It is important to keep **primary keys as small as possible**. 

### Tenant Scoping

How do we ensure that all application queries mandatorily filter on the `tenant_id` column? While there are approaches to implementing shared schema model in the application ([django](https://books.agiliq.com/projects/django-multi-tenant/en/latest/shared-database-shared-schema.html), [hibernate filters](https://callistaenterprise.se/blogg/teknik/2020/10/17/multi-tenancy-with-spring-boot-part5/), [hibernate 6 @TenantId](https://callistaenterprise.se/blogg/teknik/2023/05/22/multi-tenancy-with-spring-boot-part8/)), they all have limitations. For example, an implementation using hibernate would not handle native sql queries or when some other data access library like [JooQ](https://www.jooq.org/) is used by the application.

Managing multi-tenancy centrally as part of the database would be much better. Postgres (*since version 9.5*) supports [Row Level Security](https://www.postgresql.org/docs/current/ddl-rowsecurity.html). With Postgres RLS,  it is [easy to implement](https://www.crunchydata.com/blog/row-level-security-for-tenants-in-postgres) [shared schema multi-tenancy](https://docs.aws.amazon.com/prescriptive-guidance/latest/saas-multitenant-managed-postgresql/rls.html) by creating a policy for each table which filters rows of a single tenant.

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

Also note the `with check option` clause. This ensures that DML statements cannot insert/update rows that will violate the view definition criteria. i.e. `tenant_id` cannot be set to any other value other than `currentTenant` value.

#### Restricting access to base tables

With the updatable views, when application accesses data via the views, the tenant scoping is done automatically based on the `currentTenant` session variable. But, what if the application directly accesses the underlying table? In that case, we run the risk of application reading/writing data of all tenants.

To solve this, we create two MySQL roles `data_owner` and `app_rw`. We create  two separate schemas - `data` and `app`. Tables are created in `data`  schema and views in `app` schema. Access to `data` schema is granted to `data_owner` role and `app` schema to `app_rw` role.

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

Create the views in `app` schema. Note that we specify DEFINER as `data_owner` so that the view has access to the underlying table although the role `app_rw` may not have access to the tables.

```sql
CREATE DEFINER=data_owner ALGORITHM=MERGE VIEW app.department AS SELECT * FROM data.department WHERE tenant_id = data.current_tenant() WITH CHECK OPTION;
CREATE DEFINER=data_owner ALGORITHM=MERGE VIEW app.user AS SELECT * FROM data.user WHERE tenant_id = data.current_tenant() WITH CHECK OPTION;
```

Optionally, we could also set up before insert triggers on the base tables so that, the `tenant_id` is automatically populated. This way, the application does not have to do anything related to multi-tenancy and everything is handled in MySQL itself.
```sql
CREATE TRIGGER data.department_tenant BEFORE INSERT ON Department FOR EACH ROW SET new.tenant_id = data.current_tenant();
CREATE TRIGGER data.user_tenant BEFORE INSERT ON User FOR EACH ROW SET new.tenant_id = data.current_tenant();
```

### Example spring boot application

Applications just have to set the current tenant's id  as a session variable. We can define a custom datasource `MTDataSource` which acts as a wrapper for the underlying data source. When application invokes `getConnection`, it gets the connection from acutal datasource and sets the current tenant id.

```kotlin
class MTDataSource(val actualDS: DataSource) : DataSource by actualDS {

    override fun getConnection(): Connection {
        var connection = actualDS.connection
        var statement: PreparedStatement? = null
        try {
            statement = connection.prepareStatement("SET @currentTenant = ?")
            val currentTenantId = TenantContext.getCurrentTenantId()
            if(currentTenantId != null) {
                statement.setInt(1, currentTenantId)
                statement.execute()
            }
        } finally {
            statement?.close()
        }
        return MTConnection(connection)
    }
}
```

The datasource returns a `MTConnection` instance, which is a wrapper for the connection returned by the actual data source. It resets the `currentTenant` when the connection is closed.
```kotlin
class MTConnection(val connection: Connection) : Connection by connection{
    override fun close() {
        var statement:Statement? = null
            try{
            statement = connection.createStatement()
            statement.execute("SET @currentTenant = NULL")
        } finally {
            try {
                statement?.close()
            } catch (e: Exception){
                e.printStackTrace()
            }
            connection.close()
        }
    }

}
```
With this approach, the application models, repositories are defined as if it is a single tenant application.  A full example implementation using spring boot is available at [multi-tenant-ticketing-sample](https://github.com/Nandakumar-M/multi-tenant-ticketing-sample).


### Performance

Our multi-tenancy architecture is essentially built on MySQL views. How are views processed by MySQL? 

There are two [view processing algorithms](https://dev.mysql.com/doc/refman/8.0/en/view-algorithms.html) in MySQL.
1. MERGE
2. TEMPTABLE

With `MERGE` algorithm (which we specified in our create view statements), the criteria of the view definition is merged into the outer query. For example, a query 
```sql
select * from app.department where id = 1;
```
that references the view `app.department` gets converted into 
```sql
select * from data.department where tenant_id = current_tenant() and id = 1;
```
The MySQL [Optimizing Derived Tables, View References, and Common Table Expressions with Merging or Materialization](https://dev.mysql.com/doc/refman/8.0/en/derived-table-optimization.html) page says the following.

> The optimizer handles derived tables, view references, and common table expressions the same way: It avoids unnecessary materialization whenever possible, which enables pushing down conditions from the outer query to derived tables and produces more efficient execution plans. (For an example, see Section 10.2.2.2, “Optimizing Subqueries with Materialization”.)
> 
> If merging would result in an outer query block that references more than **61 base tables**, the optimizer chooses materialization instead.

Although MySQL does not support creating indexes on views, with MERGE algorithm, the indexes on the base tables are used by the query planner.

By populating the example application database with dummy data for several hundred tenants, we can check the query plans and verify that the primary and secondary indexes are getting utilized.

```sql
mysql> set @currTenant=229;
Query OK, 0 rows affected (0.01 sec)

mysql> explain analyze select * from ticket where priorityId =1 and impactId=1 and categoryId=1;
+--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| EXPLAIN                                                                                                                                                                                                                              |
+--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| -> Filter: ((data.ticket.tenantId = <cache>(data.current_tenant())) and (data.ticket.categoryId = 1) and (data.ticket.impactId = 1) and (data.ticket.priorityId = 1))  (cost=4371 rows=329) (actual time=2.32..139 rows=82 loops=1)
    -> Intersect rows sorted by row ID  (cost=4371 rows=3287) (actual time=0.498..135 rows=20583 loops=1)
        -> Index range scan on ticket using ticket_impact_fk over (tenantId = 229 AND impactId = 1)  (cost=171 rows=109576) (actual time=0.32..14.6 rows=56797 loops=1)
        -> Index range scan on ticket using ticket_priority_fk over (tenantId = 229 AND priorityId = 1)  (cost=707 rows=456146) (actual time=0.12..48.3 rows=258350 loops=1)
|
+--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
1 row in set (0.15 sec)

mysql> set @currTenant=20;
Query OK, 0 rows affected (0.01 sec)


mysql> explain analyze select * from ticket where priorityId =1 and impactId=1 and categoryId=1;
+-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| EXPLAIN                                                                                                                                                                                                                           |
+-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| -> Filter: ((data.ticket.tenantId = <cache>(data.current_tenant())) and (data.ticket.categoryId = 1) and (data.ticket.impactId = 1) and (data.ticket.priorityId = 1))  (cost=10.7 rows=0.1) (actual time=1.62..18 rows=9 loops=1)
    -> Intersect rows sorted by row ID  (cost=10.7 rows=1) (actual time=1.07..17.8 rows=74 loops=1)
        -> Index range scan on ticket using ticket_priority_fk over (tenantId = 20 AND priorityId = 1)  (cost=3.58 rows=1684) (actual time=0.617..0.955 rows=1684 loops=1)
        -> Index range scan on ticket using ticket_impact_fk over (tenantId = 20 AND impactId = 1)  (cost=7.06 rows=3914) (actual time=0.335..1.01 rows=3912 loops=1)
|
+-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
1 row in set (0.08 sec)
```

```sql
mysql> explain analyze select u1_0.userId,a1_0.accountId,am1_0.userId,am1_0.accountId,am1_0.activated,am1_0.createdTime,am1_0.businessFunctionId,am1_0.email,am1_0.name,am1_0.reportingManagerId,am1_0.siteId,a1_0.createdTime,a1_0.deletedTime,hs1_0.id,hs1_0.accountId,hs1_0.deletedTime,hs1_0.hq,hs1_0.name,hs1_0.working24x7,a1_0.name,a1_0.stageId,a1_0.updatedTime,u1_0.activated,u1_0.createdTime,d2_0.businessFunctionId,d2_0.deletedTime,h1_0.userId,h1_0.accountId,h1_0.activated,h1_0.createdTime,h1_0.businessFunctionId,h1_0.email,h1_0.name,h1_0.reportingManagerId,h1_0.siteId,d2_0.name,u1_0.email,u1_0.name,rm3_0.userId,a5_0.accountId,a5_0.accountManagerId,a5_0.createdTime,a5_0.deletedTime,a5_0.hqSiteId,a5_0.name,a5_0.stageId,a5_0.updatedTime,rm3_0.activated,rm3_0.createdTime,d4_0.businessFunctionId,d4_0.deletedTime,d4_0.headId,d4_0.name,rm3_0.email,rm3_0.name,rm3_0.reportingManagerId,s3_0.id,s3_0.accountId,s3_0.deletedTime,s3_0.hq,s3_0.name,s3_0.working24x7,s4_0.id,a7_0.accountId,a7_0.accountManagerId,a7_0.createdTime,a7_0.deletedTime,a7_0.hqSiteId,a7_0.name,a7_0.stageId,a7_0.updatedTime,s4_0.deletedTime,s4_0.hq,s4_0.name,s4_0.working24x7 from User u1_0 left join Account a1_0 on a1_0.accountId=u1_0.accountId left join User am1_0 on am1_0.userId=a1_0.accountManagerId left join Site hs1_0 on hs1_0.id=a1_0.hqSiteId left join businessfunction d2_0 on d2_0.businessFunctionId=u1_0.businessFunctionId left join User h1_0 on h1_0.userId=d2_0.headId left join User rm3_0 on rm3_0.userId=u1_0.reportingManagerId left join Account a5_0 on a5_0.accountId=rm3_0.accountId left join businessfunction d4_0 on d4_0.businessFunctionId=rm3_0.businessFunctionId left join Site s3_0 on s3_0.id=rm3_0.siteId left join Site s4_0 on s4_0.id=u1_0.siteId left join Account a7_0 on a7_0.accountId=s4_0.accountId where u1_0.userId=205;
+------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| EXPLAIN                                                                                                                                                                                                                            |
+------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| -> Nested loop left join  (cost=10.5 rows=1) (actual time=0.302..0.304 rows=1 loops=1)
    -> Nested loop left join  (cost=9.36 rows=1) (actual time=0.283..0.285 rows=1 loops=1)
        -> Nested loop left join  (cost=8.27 rows=1) (actual time=0.282..0.284 rows=1 loops=1)
            -> Nested loop left join  (cost=7.18 rows=1) (actual time=0.277..0.278 rows=1 loops=1)
                -> Nested loop left join  (cost=6.83 rows=1) (actual time=0.248..0.249 rows=1 loops=1)
                    -> Nested loop left join  (cost=5.74 rows=1) (actual time=0.225..0.226 rows=1 loops=1)
                        -> Nested loop left join  (cost=4.67 rows=1) (actual time=0.2..0.201 rows=1 loops=1)
                            -> Nested loop left join  (cost=3.6 rows=1) (actual time=0.175..0.176 rows=1 loops=1)
                                -> Nested loop left join  (cost=3.25 rows=1) (actual time=0.139..0.139 rows=1 loops=1)
                                    -> Nested loop left join  (cost=2.16 rows=1) (actual time=0.0998..0.1 rows=1 loops=1)
                                        -> Nested loop left join  (cost=1.1 rows=1) (actual time=0.0648..0.0651 rows=1 loops=1)
                                            -> Rows fetched before execution  (cost=0..0 rows=1) (actual time=84e-6..126e-6 rows=1 loops=1)
                                            -> Single-row index lookup on account using PRIMARY (tenantId=data.current_tenant(), accountId='16')  (cost=1.1 rows=1) (actual time=0.0544..0.0544 rows=1 loops=1)
                                        -> Single-row index lookup on user using PRIMARY (tenantId=data.current_tenant(), userId=data.`account`.accountManagerId)  (cost=1.07 rows=1) (actual time=0.0343..0.0344 rows=1 loops=1)
                                    -> Single-row index lookup on site using PRIMARY (tenantId=data.current_tenant(), id=data.`account`.hqSiteId)  (cost=1.09 rows=1) (actual time=0.0384..0.0384 rows=1 loops=1)
                                -> Single-row index lookup on businessfunction using PRIMARY (tenantId=data.current_tenant(), businessFunctionId='75')  (cost=0.35 rows=1) (actual time=0.0364..0.0364 rows=1 loops=1)
                            -> Single-row index lookup on user using PRIMARY (tenantId=data.current_tenant(), userId=data.businessfunction.headId)  (cost=1.07 rows=1) (actual time=0.0246..0.0247 rows=1 loops=1)
                        -> Single-row index lookup on user using PRIMARY (tenantId=data.current_tenant(), userId='45')  (cost=1.07 rows=1) (actual time=0.0241..0.0242 rows=1 loops=1)
                    -> Single-row index lookup on account using PRIMARY (tenantId=data.current_tenant(), accountId=data.`user`.accountId)  (cost=1.1 rows=1) (actual time=0.0226..0.0226 rows=1 loops=1)
                -> Single-row index lookup on businessfunction using PRIMARY (tenantId=data.current_tenant(), businessFunctionId=data.`user`.businessFunctionId)  (cost=0.35 rows=1) (actual time=0.0288..0.0289 rows=1 loops=1)
            -> Single-row index lookup on site using PRIMARY (tenantId=data.current_tenant(), id=data.`user`.siteId)  (cost=1.09 rows=1) (actual time=0.00442..0.00442 rows=0 loops=1)
        -> Single-row index lookup on site using PRIMARY (tenantId=data.current_tenant(), id=NULL)  (cost=1.09 rows=1) (actual time=0.00125..0.00125 rows=0 loops=1)
    -> Single-row index lookup on account using PRIMARY (tenantId=data.current_tenant(), accountId=data.site.accountId)  (cost=1.1 rows=1) (actual time=0.00971..0.00971 rows=0 loops=1)
|
+------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
1 row in set (0.03 sec)
```


### Limitations
There are a few limitations in our approach.

1. Foreign key action `ON DELETE SET NULL` sets all the FK columns to `NULL` in case of composite FKs. i.e. When a user referenced in department table using (`tenant_id`, `head_id`) is deleted, MySQL tries to set both `tenant_id` and `head_id` to `NULL` which violates the NOT NULL constraint on the `tenant_id` column. We can workaround this by creating BEFORE DELETE triggers on user table.
2. Since there are no indexes for views, application queries cannot have index hints.
3. Statement-based binary logging and replication cannot be used as our queries uses `current_tenant` function which relies on session variable. Row based binary logging and replication is possible.

## Conclusion

We have seen an efficient and scalable approach to implementing multi-tenancy with MySQL. This approach has the added advantage that it can be employed in the case of an application that currently uses separate schema approach to multi-tenancy with minimal code changes.