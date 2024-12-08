# Data Integrity

- [Column Types](#column-types)
- [Constraints](#constraints)
- [Foreign Key](#foreign-key)
  - [Not Null Constraint](#not-null-constraint)
  - [Primary Key](#primary-key)
  - [Unique Constraint](#unique-constraint)
  - [Check Constraint](#check-constraint)
- [Adding constraints to existing tables](#adding-constraints-to-existing-tables)
- [Crucial for data integrity](#crucial-for-data-integrity)

## Column Types

```sql
INSERT INTO users (id, name, active, date_joined) VALUES (1, 'John Doe', 'not valid' ,'2023-08-01');
```

Cannon insert invalid values

```bash
db1=# INSERT INTO users (id, name, active, date_joined) VALUES (1, 'John Doe', 'not valid' ,'2023-08-01');
ERROR:  invalid input syntax for type boolean: "not valid"
LINE 1: ...name, active, date_joined) VALUES (1, 'John Doe', 'not valid...
```

Common Types

- varchar(N), text
- smallint, integer, bigint
- decimal, real, double precision
- date, timestamp, timestamptz
- bytea
- boolean

Special types

- uuid
- json, jsonb
- hstore
- range
- array

ref https://www.postgresql.org/docs/current/datatype.html

Set restrictions on columns

```bash
db1=# ALTER TABLE users ALTER name TYPE varchar(50);
ALTER TABLE
db1=# \d users;
                                   Table "public.users"
   Column    |         Type          | Collation | Nullable |           Default
-------------+-----------------------+-----------+----------+------------------------------
 id          | integer               |           | not null | generated always as identity
 active      | boolean               |           | not null | true
 name        | character varying(50) |           | not null |
 date_joined | date                  |           |          |
Indexes:
    "users_pkey" PRIMARY KEY, btree (id)

db1=#

db1=# INSERT INTO users (name) VALUES ('this is very long name that is not valid');
ERROR:  value too long for type character varying(20)
```

## Constraints

- Maintain data integrity
- Keeps the data clean
- Complements types

### Not Null Constraint

- Make fields required
- Null is a special values that indicated "missing value"

Add a "not null" constraint on an existing table

```bash
db1=# ALTER TABLE users ALTER active SET NOT NULL;
ALTER TABLE
db1=# \d users
                                   Table "public.users"
   Column    |         Type          | Collation | Nullable |           Default
-------------+-----------------------+-----------+----------+------------------------------
 id          | integer               |           | not null | generated always as identity
 active      | boolean               |           | not null | true
 name        | character varying(20) |           | not null |
 date_joined | date                  |           |          |
Indexes:
    "users_pkey" PRIMARY KEY, btree (id)

db1=#

db1=# INSERT INTO users (name, active) VALUES ('foo', NULL);
ERROR:  null value in column "active" of relation "users" violates not-null constraint
DETAIL:  Failing row contains (18, null, foo, null).
db1=#
```

### Primary Key

Set of fields that uniquely identify each row in a table

- must be unique
- must contain a value
- a table can only have one primary key
- a table does not have to have a primary key
- can consist of multiple fields

```bash
db1=# \d users;
                                   Table "public.users"
   Column    |         Type          | Collation | Nullable |           Default
-------------+-----------------------+-----------+----------+------------------------------
 id          | integer               |           | not null | generated always as identity
 active      | boolean               |           | not null | true
 name        | character varying(20) |           | not null |
 date_joined | date                  |           |          |
Indexes:
    "users_pkey" PRIMARY KEY, btree (id)
```

- create a unique index to enforce uniqueness
- mark the field as not null

Auto incrementing primary key

```sql
CREATE TABLE users (
    id INT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    name VARCHAR(20) NOT NULL,
    active BOOLEAN NOT NULL,
    date_joined DATE
);
```

- create a sequence
- automatically populate with next value

Control how user-specified values are handled

```sql
GENERATED {ALWAYS | BY DEFAULT } AS IDENTITY
```

- Always: prevent from setting values explicitly
- By Default: allow setting values explicitly

```bash
db1=# INSERT INTO users (id, name) VALUES (999, 'Tom');
ERROR:  cannot insert a non-DEFAULT value into column "id"
DETAIL:  Column "id" is an identity column defined as GENERATED ALWAYS.
HINT:  Use OVERRIDING SYSTEM VALUE to override.
db1=#
```

### Unique Constraint

- ensure one or more field are uniqeu
- enforced using a unique B-Tree index

Name must be unique

```bash
db1=# ALTER TABLE users ADD CONSTRAINT users_name_unique UNIQUE(name);
ALTER TABLE

db1=# \d users
                                   Table "public.users"
   Column    |         Type          | Collation | Nullable |           Default
-------------+-----------------------+-----------+----------+------------------------------
 id          | integer               |           | not null | generated always as identity
 active      | boolean               |           | not null | true
 name        | character varying(20) |           | not null |
 date_joined | date                  |           |          |
Indexes:
    "users_pkey" PRIMARY KEY, btree (id)
    "users_name_unique" UNIQUE CONSTRAINT, btree (name)

db1=#

db1=# INSERT INTO users (name) VALUES ('hoang');
INSERT 0 1
db1=# INSERT INTO users (name) VALUES ('hoang');
ERROR:  duplicate key value violates unique constraint "users_name_unique"
DETAIL:  Key (name)=(hoang) already exists.
db1=#
```

Name is not really uniqeu, so we can drop the constraint

```bash
db1=# ALTER TABLE users DROP CONSTRAINT users_name_unique;
ALTER TABLE
```

### Check Constraint

Enforce special validation logic

Name must contain at least one whitespace

```bash
db1=# ALTER TABLE users ADD CONSTRAINT name_must_contain_whitespace CHECK (name LIKE '% %');
ERROR:  check constraint "name_must_contain_whitespace" of relation "users" is violated by some row

db1=# SELECT * FROM users WHERE name NOT LIKE '% %';
 id | active | name  | date_joined
----+--------+-------+-------------
  1 | t      | Hoang |
  2 | t      | User1 |
 19 | t      | hoang |
(3 rows)

db1=#

db1=# UPDATE users
db1-# SET name = CONCAT(name, ' test')
db1-# WHERE name NOT LIKE '% %';
UPDATE 3

db1=# SELECT * FROM users WHERE name NOT LIKE '% %';
 id | active | name | date_joined
----+--------+------+-------------
(0 rows)

db1=# ALTER TABLE users ADD CONSTRAINT name_must_contain_whitespace CHECK (name LIKE '% %');
ALTER TABLE
db1=# \d users
                                   Table "public.users"
   Column    |         Type          | Collation | Nullable |           Default
-------------+-----------------------+-----------+----------+------------------------------
 id          | integer               |           | not null | generated always as identity
 active      | boolean               |           | not null | true
 name        | character varying(20) |           | not null |
 date_joined | date                  |           |          |
Indexes:
    "users_pkey" PRIMARY KEY, btree (id)
Check constraints:
    "name_must_contain_whitespace" CHECK (name::text ~~ '% %'::text)

db1=#

db1=# INSERT INTO users (name) VALUES ('le');
ERROR:  new row for relation "users" violates check constraint "name_must_contain_whitespace"
DETAIL:  Failing row contains (21, t, le, null).
```

## Foreign Key

Add a foreign key between users and their credentials

```bash
db1=# \d restricted.credentials
                      Table "restricted.credentials"
  Column  |  Type   | Collation | Nullable |           Default
----------+---------+-----------+----------+------------------------------
 id       | integer |           | not null | generated always as identity
 user_id  | integer |           | not null |
 password | text    |           | not null |
Indexes:
    "credentials_pkey" PRIMARY KEY, btree (id)

db1=# \d users
                                   Table "public.users"
   Column    |         Type          | Collation | Nullable |           Default
-------------+-----------------------+-----------+----------+------------------------------
 id          | integer               |           | not null | generated always as identity
 active      | boolean               |           | not null | true
 name        | character varying(20) |           | not null |
 date_joined | date                  |           |          |
Indexes:
    "users_pkey" PRIMARY KEY, btree (id)
Check constraints:
    "name_must_contain_whitespace" CHECK (name::text ~~ '% %'::text)

db1=#

db1=# ALTER TABLE restricted.credentials ADD CONSTRAINT user_fk
db1-# FOREIGN KEY (user_id) REFERENCES users(id)
db1-# ON DELETE CASCADE;
ALTER TABLE

db1=# \d restricted.credentials
                      Table "restricted.credentials"
  Column  |  Type   | Collation | Nullable |           Default
----------+---------+-----------+----------+------------------------------
 id       | integer |           | not null | generated always as identity
 user_id  | integer |           | not null |
 password | text    |           | not null |
Indexes:
    "credentials_pkey" PRIMARY KEY, btree (id)
Foreign-key constraints:
    "user_fk" FOREIGN KEY (user_id) REFERENCES users(id) ON DELETE CASCADE

db1=# \d users
                                   Table "public.users"
   Column    |         Type          | Collation | Nullable |           Default
-------------+-----------------------+-----------+----------+------------------------------
 id          | integer               |           | not null | generated always as identity
 active      | boolean               |           | not null | true
 name        | character varying(20) |           | not null |
 date_joined | date                  |           |          |
Indexes:
    "users_pkey" PRIMARY KEY, btree (id)
Check constraints:
    "name_must_contain_whitespace" CHECK (name::text ~~ '% %'::text)
Referenced by:
    TABLE "restricted.credentials" CONSTRAINT "user_fk" FOREIGN KEY (user_id) REFERENCES users(id) ON DELETE CASCADE

db1=#

db1=# INSERT INTO restricted.credentials (user_id, password) VALUES (99, 'test123');
ERROR:  insert or update on table "credentials" violates foreign key constraint "user_fk"
DETAIL:  Key (user_id)=(99) is not present in table "users".
db1=#
```

How deletes are handled in child tables

- CASCADE: When a user is deleted from users table, delete the associated credentials in the table credentials
- SET NULL: When a user is deleted from users table, set the user_id to null in the table credentials
- RESTRICT: When a user is deleted from users table, do not allow the deletion

It is easier to remove constraints than add them

**Always start with constraints**

## Adding constraints to existing tables

What happening under the hood:

- lock table
- enforce constraint for new rows
- validate the existing data
- release lock

Cant lock the system for a long time then what can we do?

Split the process into two parts:

1. Enforce constraints for new rows

```sql
ALTER TABLE users ADD CONSTRAINT name_must_contain_whitespace CHECK (name LIKE '% %') NOT VALID;
```

NOT VALID: Don't validate the existing data

2. Validate the existing data

```sql
ALTER TABLE users VALIDATE CONSTRAINT name_must_contain_whitespace;
```

- It is easier to change constraints than column data types
- `VARCHAR(N)` -> `TEXT` with a check constraint on length

Replace `VARCHAR(N)` with `TEXT`

```sql
CREATE TABLE users (
    id INT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    -- name VARCHAR(20) NOT NULL,
    active BOOLEAN NOT NULL,
    date_joined DATE,
    name TEXT NOT NULL,
    CONSTRAINT name_must_contain_whitespace CHECK (name LIKE '% %')
);
```

This will make it easier to check in the future

## Crucial for data integrity

- database is the source of truth
- database is the last line of defense
- data integrity is crucial

Constraints are good... use them!!!
