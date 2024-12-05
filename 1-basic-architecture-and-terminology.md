# Basic Architecture and Terminology

- [Access Postgres](#access-postgres)
- [List out database](#list-out-database)
- [Create database](#create-database)
- [Create table](#create-table)
- [Delete Table](#delete-table)
- [Inspect a table](#inspect-a-table)
- [List all table](#list-all-table)
- [Searching for table](#searching-for-table)
- [Create data](#create-data)
- [Unlogged Table](#unlogged-table)
- [Create a view](#create-a-view)

## Access Postgres

```bash
psql -U postgres
psql -U postgres -d mydb
psql -U postgres -h localhost
psql -U postgres -p 5432
psql -U postgres -d mydb -h localhost -p 5432
```

## List out database

```bash
\l
```

![alt text](./images/basic-architecture-and-terminology/image.png)

## Create database

```bash
CREATE DATABASE db1 OWNER postgres;

\c db1
```

output

```
postgres=# CREATE DATABASE db1 OWNER postgres;
CREATE DATABASE
postgres=# \c db1
You are now connected to database "db1" as user "postgres".
db1=#
```


## Create table

```bash
CREATE TABLE users (
    id INT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    active BOOLEAN NOT NULL DEFAULT TRUE,
    name TEXT NOT NULL
);
```

Create a table only if it does not already exist

```bash
CREATE TABLE IF NOT EXISTS users (
    id INT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    active BOOLEAN NOT NULL DEFAULT TRUE,
    name TEXT NOT NULL
);
```

## Delete Table

```bash
DROP TABLE users;
```

if the table does not exist, no error is raises

```bash
DROP TABLE IF EXISTS users;
```

## Inspect a table

```bash
\d users
```

![alt text](./images/basic-architecture-and-terminology/image-1.png)

## List all table

```bash
\dt
```

output

```bash
db1=# \dt
         List of relations
 Schema | Name  | Type  |  Owner
--------+-------+-------+----------
 public | users | table | postgres
(1 row)
```

## Searching for table

```bash
\dt us*
```

output

```bash
db1=# \dt us*
         List of relations
 Schema | Name  | Type  |  Owner
--------+-------+-------+----------
 public | users | table | postgres
(1 row)

db1=# \dt foo*
Did not find any relation named "foo*".
db1=#
```

## Create data

```bash
INSERT INTO users (name) 
VALUES ('Hoang'), 
       ('User1');
```

output


```bash
db1=# INSERT INTO users (name)
VALUES ('Hoang'),
       ('User1');
INSERT 0 2
db1=# select * from users;
 id | active | name
----+--------+-------
  1 | t      | Hoang
  2 | t      | User1
(2 rows)

db1=#
```

## Unlogged Table

Data written to unlogged table is not written to the WAL

```bash
CREATE UNLOGGED TABLE users (
    id INT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    name TEXT NOT NULL,
    active BOOLEAN NOT NULL DEFAULT TRUE
);
```

This unlogged table is speed up ETL.

ETL stands for Extract, Transform, Load. It is a process used in data management, specifically in data warehousing, to handle the movement and transformation of data from one or more sources to a destination system, usually for analytics or reporting purposes.

## Create a view

```bash
CREATE VIEW active_users AS
SELECT * 
FROM users
WHERE active;
```

We also have temporary view

```bash
CREATE TEMPORARY VIEW active_users AS
SELECT * 
FROM users
WHERE active;
```

Note: the temporary view is session-specific. Once you close the database connection or session, it will be dropped automatically.