# Schemas and the Search Path

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
