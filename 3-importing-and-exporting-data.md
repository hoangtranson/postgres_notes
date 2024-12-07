# Importing and Exporting Data

## Copy command

- Export data to text, CSV, or binary format
- Import data from text, CSV, or binary format to tables
- Produce reports from query results

```bash
postgres=# \h COPY
Command:     COPY
Description: copy data between a file and a table
Syntax:
COPY table_name [ ( column_name [, ...] ) ]
    FROM { 'filename' | PROGRAM 'command' | STDIN }
    [ [ WITH ] ( option [, ...] ) ]
    [ WHERE condition ]

COPY { table_name [ ( column_name [, ...] ) ] | ( query ) }
    TO { 'filename' | PROGRAM 'command' | STDOUT }
    [ [ WITH ] ( option [, ...] ) ]

where option can be one of:

    FORMAT format_name
    FREEZE [ boolean ]
    DELIMITER 'delimiter_character'
    NULL 'null_string'
    HEADER [ boolean ]
    QUOTE 'quote_character'
    ESCAPE 'escape_character'
    FORCE_QUOTE { ( column_name [, ...] ) | * }
    FORCE_NOT_NULL ( column_name [, ...] )
    FORCE_NULL ( column_name [, ...] )
    ENCODING 'encoding_name'

URL: https://www.postgresql.org/docs/14/sql-copy.html

postgres=#
```

## Import from CSV

Execute OS command from psql using `\!`

```bash
db1=# \! ls
1-basic-architecture-and-terminology.md	3-importing-and-exporting-data.md	data
2-schemas-and-the-search-path.md	README.md				images
db1=# \! cat data/users.csv
name,active,date_joined
user1,true,2018-01-01
user2,true,2018-01-01
user3,true,2018-01-01

db1=#
```

Psql treat first row as data but they are headers

```bash
db1=# \COPY users FROM data/users.csv
ERROR:  invalid input syntax for type integer: "name,active,date_joined"
CONTEXT:  COPY users, line 1, column id: "name,active,date_joined"
db1=#
```

```bash
db1=# \COPY users FROM data/users.csv WITH CSV HEADER
ERROR:  missing data for column "active"
CONTEXT:  COPY users, line 4: ""
db1=#
```

Final command line

```bash
\COPY users(active, name, date_joined)
FROM 'data/users.csv'
WITH CSV HEADER;
```

Output

```bash
db1=# \COPY users(active, name, date_joined)
FROM 'data/users.csv'
WITH CSV HEADER;
COPY 2
db1=# select * from users;
 id | active |  name  | date_joined
----+--------+--------+-------------
  1 | t      | Hoang  |
  2 | t      | User1  |
  3 | f      | User 2 |
  4 | f      | User 3 |
 15 | t      | User 5 | 2024-12-12
 16 | f      | User 6 | 2024-12-12
(6 rows)

db1=#
```

## Export table to CSV

```bash
db1=# \COPY users TO data/users1.csv WITH CSV HEADER
COPY 6
db1=#
```

export by query

```bash
db1=# \COPY (SELECT * FROM users where active) TO data/users1.csv WITH CSV HEADER
COPY 3
db1=#
```

## Report

```bash
db1=# \d restricted.credentials;
                      Table "restricted.credentials"
  Column  |  Type   | Collation | Nullable |           Default
----------+---------+-----------+----------+------------------------------
 id       | integer |           | not null | generated always as identity
 user_id  | integer |           | not null |
 password | text    |           | not null |
Indexes:
    "credentials_pkey" PRIMARY KEY, btree (id)

db1=#


db1=# \d users
                            Table "public.users"
   Column    |  Type   | Collation | Nullable |           Default
-------------+---------+-----------+----------+------------------------------
 id          | integer |           | not null | generated always as identity
 active      | boolean |           | not null | true
 name        | text    |           | not null |
 date_joined | date    |           |          |
Indexes:
    "users_pkey" PRIMARY KEY, btree (id)

db1=#
```

So we want to export report for this command

```sql
select u.id, u.name, count(*) as credentials
from users u
left join restricted.credentials c on u.id = c.user_id
where u.active
group by 1, 2;
```

Solution is that using temporary view

```sql
CREATE TEMPORARY VIEW report AS
select u.id, u.name, count(*) as credentials
from users u
left join restricted.credentials c on u.id = c.user_id
where u.active
group by 1, 2;
```

then it look like

```bash
db1=# CREATE TEMPORARY VIEW report AS
select u.id, u.name, count(*) as credentials
from users u
left join restricted.credentials c on u.id = c.user_id
where u.active
group by 1, 2;
CREATE VIEW

db1=# \COPY (SELECT * FROM report) TO STDOUT WITH CSV HEADER
id,name,credentials
2,User1,1
15,User 5,1
1,Hoang,1
```
