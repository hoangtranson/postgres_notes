# Schemas and the Search Path

- [Create a schema](#create-a-schema)
- [Create table in a schema](#create-table-in-a-schema)
- [Referencing object by name in schema](#referencing-object-by-name-in-schema)
- [Multi-tenancy](#multi-tenancy)

What is Schema?

- A namespace within the database
- Database can use multiple schemas
- Can have object with similar names in different schemas
- The default schema is called "public"

## Create a schema

```sql
CREATE SCHEMA restricted;
```

## Create table in a schema

```sql
CREATE TABLE restricted.credentials (
    id INT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    user_id INT NOT NULL,
    password TEXT NOT NULL
);
```

## Referencing object by name in schema

1. database.schema.table

- Fully qualified name including the database, schema, and table.
- Use Case: Typically used in cross-database queries (e.g., when using extensions like dblink or foreign data wrappers).

```sql
SELECT * FROM db1.restricted.credentials;
```

2. schema.table

- Refers to a table in a specific schema within the current database.
- Use Case: Use this format when you want to explicitly reference a table in a specific schema, bypassing the search_path.

```sql
SELECT * FROM restricted.credentials;
```

3. table

- Shortest form, referencing a table without explicitly specifying the schema.
- Use Case: Relies on the `search_path` configuration to determine which schema to look in.

```sql
SELECT * FROM credentials;
```

- PostgreSQL checks schemas in the order listed in the search_path.
- If there are multiple tables with the same name in different schemas, the one in the first schema in the search_path is used.

Show search path

```bash
db1=# SHOW search_path;
   search_path
-----------------
 "$user", public
(1 row)

db1=#
```

Ex:

User App references to table credentials, Postgres will search:

- app.credentials
- public.credentials

Using unqualified name

```bash
db1=# SELECT * FROM credentials;
ERROR:  relation "credentials" does not exist
LINE 1: SELECT * FROM credentials;
                      ^
db1=#
```

So we need to set again search path

```bash
db1=# SET search_path TO restricted, "$user", public;
SET
db1=#
```

Check search path

```bash
db1=# SHOW search_path;
         search_path
-----------------------------
 restricted, "$user", public
(1 row)

db1=#
```

Query table credentials again

```bash
db1=# SELECT * FROM credentials;
 id | user_id | password
----+---------+----------
(0 rows)

db1=#
```

## Multi-tenancy

Implement multi-tenancy using schemas in PostgreSQL where each tenant (e.g., tenantA, tenantB) has their own schema, and some tables are shared across all tenants (e.g., order in a shared schema and category and product in public)

1. Create the Shared and Public Schemas

```sql
-- Create the shared schema
CREATE SCHEMA shared;

-- Create the shared `order` table
CREATE TABLE shared.orders (
    id SERIAL PRIMARY KEY,
    tenant_id TEXT NOT NULL, -- To track which tenant owns the order
    product_id INT NOT NULL,
    quantity INT NOT NULL,
    order_date TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Ensure the public schema has `category` and `product` tables
CREATE TABLE public.categories (
    id SERIAL PRIMARY KEY,
    name TEXT NOT NULL
);

CREATE TABLE public.products (
    id SERIAL PRIMARY KEY,
    name TEXT NOT NULL,
    category_id INT REFERENCES public.categories(id)
);
```

2. Create Tenant-Specific Schemas

```sql
-- Create schemas for each tenant
CREATE SCHEMA tenantA;
CREATE SCHEMA tenantB;

-- Example of creating a tenant-specific table (if needed)
CREATE TABLE tenantA.customers (
    id SERIAL PRIMARY KEY,
    name TEXT NOT NULL,
    email TEXT UNIQUE NOT NULL
);

CREATE TABLE tenantB.customers (
    id SERIAL PRIMARY KEY,
    name TEXT NOT NULL,
    email TEXT UNIQUE NOT NULL
);
```

3. Access Control

```sql
-- Revoke default permissions from all users
REVOKE ALL ON SCHEMA tenantA, tenantB, shared FROM PUBLIC;

-- Grant permissions to specific roles
GRANT USAGE ON SCHEMA tenantA TO tenantA_role;
GRANT USAGE ON SCHEMA tenantB TO tenantB_role;

-- Grant SELECT/INSERT/UPDATE/DELETE privileges on tables
GRANT SELECT, INSERT, UPDATE, DELETE ON ALL TABLES IN SCHEMA shared TO tenantA_role, tenantB_role;
GRANT SELECT, INSERT, UPDATE, DELETE ON ALL TABLES IN SCHEMA public TO tenantA_role, tenantB_role;
```

4. Set the Search Path for Each Tenant

```sql
-- Set search_path for Tenant A
SET search_path TO tenantA, shared, public;

-- Set search_path for Tenant B
SET search_path TO tenantB, shared, public;
```

For set search path, we have serveral options:

- Temporary Search Path (Session-Level)

```sql
-- Set the search path for Tenant A in the current session
SET search_path TO tenantA, shared, public;

-- Verify the search path
SHOW search_path;

-- Execute queries targeting the tenantA schema first
SELECT * FROM customers; -- This will look in tenantA.customers
```

- Persistent Search Path (User-Level)

```sql
-- Set the search path for a specific tenant user
ALTER ROLE tenantA_role SET search_path TO tenantA, shared, public;

-- For Tenant B
ALTER ROLE tenantB_role SET search_path TO tenantB, shared, public;
```

- Using Application Logic for Multi-Tenancy

```js
const { Pool } = require("pg");

// Create a database connection pool
const pool = new Pool({
  user: "dbuser",
  host: "localhost",
  database: "mydb",
  password: "password",
  port: 5432,
});

async function setSearchPath(tenant) {
  const client = await pool.connect();
  try {
    // Dynamically set the search path
    await client.query(`SET search_path TO ${tenant}, shared, public`);
    // Execute tenant-specific queries
    const res = await client.query("SELECT * FROM customers");
    console.log(res.rows);
  } finally {
    client.release();
  }
}

// Example usage
setSearchPath("tenantA");
setSearchPath("tenantB");
```

- Default Search Path for All Users (Database-Level)

```sql
ALTER DATABASE mydb SET search_path TO public, shared;
```

## Migrations using Schemas

### Example 1 - Long Migrations

- Use a user-specific schema to "redirect" queries during migration.
- Create a view in the user schema that mimics the production table structure but serves as a placeholder while the migration/backfill is being completed.
- Use the search_path to prioritize the user-specific schema temporarily, allowing the app to continue using unqualified names without disruption.

1. Schema and View Setup

```sql
-- Create a user-specific schema
CREATE SCHEMA tenantA;

-- Create a view in the schema that mirrors the production table
CREATE VIEW tenantA.orders AS
SELECT * FROM migrations.orders_staging;
```

- `migrations.orders_staging`: Temporary table where backfilling or migration operations occur.

2. Configure search_path

```sql
SET search_path TO tenantA, public;
```

- With this setup, unqualified names (e.g., orders) will first resolve to tenantA.orders (the view).
- Once the migration or backfill is complete, switch back to the standard public schema.

3. Workflow for Long Migrations

Before Migration:

```sql
CREATE SCHEMA migrations;

CREATE TABLE migrations.orders_staging (
    id SERIAL PRIMARY KEY,
    user_id INT NOT NULL,
    product_id INT NOT NULL,
    quantity INT NOT NULL,
    order_date TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

Populate the `migrations.orders_staging` table as part of the migration process

During Migration:

```sql
SET search_path TO tenantA, public;
```

The app continues querying orders without being aware of the migration, as the tenantA.orders view is mapped to migrations.orders_staging

After Migration:

Drop the user-specific schema and view

```sql
DROP SCHEMA tenantA CASCADE;
```

Integrate the changes into the public.orders table

```sql
INSERT INTO public.orders (id, user_id, product_id, quantity, order_date)
SELECT id, user_id, product_id, quantity, order_date
FROM migrations.orders_staging;

DROP TABLE migrations.orders_staging;
```

Reset the search_path to default

```sql
SET search_path TO public;
```

### Example 2

Add new user test1

```bash
postgres=# select * from pg_roles;
postgres=# CREATE USER test1 WITH PASSWORD 'test1';
CREATE ROLE
postgres=# GRANT CONNECT ON DATABASE db1 TO test1;
GRANT
postgres=# GRANT USAGE ON SCHEMA public TO test1;
GRANT
```

If we can get error `ERROR:  permission denied for table ...` we can try to run `GRANT SELECT ON users to test1;` from super admin user

Make a change

```
ALTER TABLE users ADD date_joined DATE;
```

Checking

```bash
db1=# \c db1 test1
You are now connected to database "db1" as user "test1".
db1=> select * from users;
 id | active |  name  | date_joined
----+--------+--------+-------------
  1 | t      | Hoang  |
  2 | t      | User1  |
  3 | f      | User 2 |
  4 | f      | User 3 |
(4 rows)

db1=> \c db1 postgres
You are now connected to database "db1" as user "postgres".
db1=# Create Schema test1;
CREATE SCHEMA
db1=# create view test1.users as select id, active, name, coalesce(date_joined, '2024-12-12') as date_joined from users;
CREATE VIEW

db1=# grant usage on schema test1 to test1;
GRANT
db1=# grant select on test1.users to test1;
GRANT

db1=# \c db1 test1
You are now connected to database "db1" as user "test1".
db1=> select * from users;
 id | active |  name  | date_joined
----+--------+--------+-------------
  1 | t      | Hoang  | 2024-12-12
  2 | t      | User1  | 2024-12-12
  3 | f      | User 2 | 2024-12-12
  4 | f      | User 3 | 2024-12-12
(4 rows)

db1=>
```

## Information Schema

```sql
SELECT * FROM pg_tables WHERE tablename ='users';
```

```bash
db1=> select * from pg_tables where tablename ='users';
 schemaname | tablename | tableowner | tablespace | hasindexes | hasrules | hastriggers | rowsecurity
------------+-----------+------------+------------+------------+----------+-------------+-------------
 public     | users     | postgres   |            | t          | f        | f           | f
(1 row)

db1=>
```

Information about index

```sql
SELECT * FROM pg_indexes WHERE tablename='users';
```

Output

```bash
db1=> select * from pg_indexes where tablename='users';
 schemaname | tablename | indexname  | tablespace |                            indexdef
------------+-----------+------------+------------+-----------------------------------------------------------------
 public     | users     | users_pkey |            | CREATE UNIQUE INDEX users_pkey ON public.users USING btree (id)
(1 row)

db1=>
```

Information about all database objects

```sql
SELECt relname, relkind FROM pg_class WHERE relname LIKE '%user%';
```

Output

```bash
db1=> select relname, relkind from pg_class where relname like '%user%';
              relname              | relkind
-----------------------------------+---------
 users_id_seq                      | S
 users_pkey                        | i
 active_users                      | v
 users_by_active_status_mv         | m
 users                             | r
 users                             | v
 pg_user_mapping_oid_index         | i
 pg_user_mapping_user_server_index | i
 pg_user_mapping                   | r
 pg_stat_xact_user_functions       | v
 pg_user                           | v
 pg_stat_xact_user_tables          | v
 pg_stat_user_tables               | v
 pg_statio_user_tables             | v
 pg_stat_user_indexes              | v
 pg_statio_user_indexes            | v
 pg_statio_user_sequences          | v
 pg_stat_user_functions            | v
 pg_user_mappings                  | v
 user_mappings                     | v
 user_defined_types                | v
 user_mapping_options              | v
 _pg_user_mappings                 | v
(23 rows)

db1=>
```

- pg_stat_activity: Information about current activity
- pg_stat_all_table: Table access statistics
- pg_stat_all_indexes: Index access statistics
- pg_stats: statistics about table columns

```bash
\set ECHO_HIDDEN on
```

to turn on hidden commands

```bash
db1=> \dt
********* QUERY **********
SELECT n.nspname as "Schema",
  c.relname as "Name",
  CASE c.relkind WHEN 'r' THEN 'table' WHEN 'v' THEN 'view' WHEN 'm' THEN 'materialized view' WHEN 'i' THEN 'index' WHEN 'S' THEN 'sequence' WHEN 's' THEN 'special' WHEN 't' THEN 'TOAST table' WHEN 'f' THEN 'foreign table' WHEN 'p' THEN 'partitioned table' WHEN 'I' THEN 'partitioned index' END as "Type",
  pg_catalog.pg_get_userbyid(c.relowner) as "Owner"
FROM pg_catalog.pg_class c
     LEFT JOIN pg_catalog.pg_namespace n ON n.oid = c.relnamespace
     LEFT JOIN pg_catalog.pg_am am ON am.oid = c.relam
WHERE c.relkind IN ('r','p','')
      AND n.nspname <> 'pg_catalog'
      AND n.nspname !~ '^pg_toast'
      AND n.nspname <> 'information_schema'
  AND pg_catalog.pg_table_is_visible(c.oid)
ORDER BY 1,2;
**************************

Did not find any relations.
db1=>
```
