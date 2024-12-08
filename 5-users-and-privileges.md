# Users and Privileges

- [Create a User](#create-a-user)
- [Privileges](#privileges)
  - [Types of privileges](#types-of-privileges)
  - [Enforce privileges](#enforce-privileges)
  - [Revoke privileges](#revoke-privileges)
  - [Check user privileges on table](#check-user-privileges-on-table)
  - [Grant privileges on schema](#grant-privileges-on-schema)
  - [Allow access to specific fields](#allow-access-to-specific-fields)
- [Roles](#roles)
  - [Create a Role](#create-a-role)
  - [Grant privileges to a role](#grant-privileges-to-a-role)
  - [Grant membership in a role](#grant-membership-in-a-role)
  - [List roles](#list-roles)

## Create a User

```bash
db1=# CREATE USER app PASSWORD 'secret';
CREATE ROLE
Time: 16.912 ms
db1=# \c db1 app
You are now connected to database "db1" as user "app".
db1=>
```

connect from terminal

```bash
psql -d db1 -U app -W
```

- d: name of database
- U: name of user
- W: prompt for password

## Privileges

### Types of privileges

- SELECT/UPDATE/DELETE/TRUNCATE on TABLE
- CONNECT on DATABASE
- CREATE on DATABASE

### Enforce privileges

```bash
db1=> \conninfo
You are connected to database "db1" as user "app" via socket in "/tmp" at port "5432".

db1=> select * from users;
ERROR:  permission denied for table users
Time: 5.244 ms
db1=>

db1=> \c db1 postgres
You are now connected to database "db1" as user "postgres".
db1=# GRANT SELECT ON users TO app;
GRANT
Time: 3.789 ms
db1=#

db1=# \c db1 app
You are now connected to database "db1" as user "app".
db1=> select * from users;
 id | active |    name    | date_joined
----+--------+------------+-------------
  3 | f      | User 2     |
  4 | f      | User 3     |
 15 | t      | User 5     | 2024-12-12
 16 | f      | User 6     | 2024-12-12
  1 | t      | Hoang test |
  2 | t      | User1 test |
 19 | t      | hoang test |
(7 rows)

Time: 3.492 ms
db1=>

db1=> INSERT INTO users (name) VALUES ('hoang test 1');
ERROR:  permission denied for table users
Time: 0.824 ms
db1=>

db1=> \c db1 postgres
You are now connected to database "db1" as user "postgres".
db1=# GRANT UPDATE, INSERT, DELETE on users TO app;
GRANT
Time: 1.805 ms

db1=# \c db1 app
You are now connected to database "db1" as user "app".
db1=> INSERT INTO users (name) VALUES ('hoang test 1');
INSERT 0 1
Time: 3.588 ms
db1=>
```

### Revoke privileges

```bash
db1=> \c db1 postgres
You are now connected to database "db1" as user "postgres".
db1=# REVOKE INSERT ON users FROM app;
REVOKE
Time: 3.898 ms
db1=#

db1=# \c db1 app
You are now connected to database "db1" as user "app".
db1=> INSERT INTO users (name) VALUES ('hoang test 2');
ERROR:  permission denied for table users
Time: 4.007 ms
db1=>
```

### Check user privileges on table

```bash
db1=> SELECT grantee, privilege_type
FROM information_schema.role_table_grants
WHERE table_name = 'users' and grantee = 'app';
 grantee | privilege_type
---------+----------------
 app     | SELECT
 app     | UPDATE
 app     | DELETE
(3 rows)

Time: 15.252 ms
db1=>
```

### Grant privileges on schema

```bash
db1=# GRANT USAGE ON SCHEMA restricted TO app;
GRANT
Time: 2.894 ms
db1=#
```

### Allow access to specific fields

```bash
db1=# GRANT SELECT (user_id) ON restricted.credentials TO app;
GRANT
Time: 1.598 ms

db1=# \c db1 app
You are now connected to database "db1" as user "app".
db1=> select * from restricted.credentials;
ERROR:  permission denied for table credentials
Time: 3.031 ms
db1=> select user_id, password from restricted.credentials;
ERROR:  permission denied for table credentials
Time: 0.999 ms
db1=> select user_id from restricted.credentials;
 user_id
---------
       1
       3
(2 rows)

Time: 0.670 ms
db1=>
```

## Roles

- similar to user
- dont have login privileges by default
- should be thought of as a group of privileges a user can be a member of

### Create a Role

```bash
db1=> \c db1 postgres
You are now connected to database "db1" as user "postgres".
db1=# CREATE ROLE analyst;
CREATE ROLE
Time: 2.457 ms
db1=#
```

### Grant priviledges to role

```bash
db1=# GRANT SELECT ON ALL TABLES IN SCHEMA public TO analyst;
GRANT
Time: 3.331 ms
db1=# GRANT UPDATE ON users TO analyst;
GRANT
Time: 2.553 ms
db1=#
```

### Grant membership in a role

```bash
db1=# CREATE USER bob;
CREATE ROLE
Time: 0.811 ms
db1=# GRANT analyst TO bob;
GRANT ROLE
Time: 3.726 ms
db1=#

db1=# \c db1 bob
You are now connected to database "db1" as user "bob".
db1=> SELECT * from users;
 id | active |     name     | date_joined
----+--------+--------------+-------------
  3 | f      | User 2       |
  4 | f      | User 3       |
 15 | t      | User 5       | 2024-12-12
 16 | f      | User 6       | 2024-12-12
  1 | t      | Hoang test   |
  2 | t      | User1 test   |
 19 | t      | hoang test   |
 22 | t      | hoang test 1 |
(8 rows)

Time: 3.009 ms
db1=>
```

### List roles

```bash
db1=> \du
                                   List of roles
 Role name |                         Attributes                         | Member of
-----------+------------------------------------------------------------+-----------
 analyst   | Cannot login                                               | {}
 app       |                                                            | {}
 bob       |                                                            | {analyst}
 hoang     | Superuser, Create role, Create DB, Replication, Bypass RLS | {}
 postgres  | Superuser, Create role, Create DB                          | {}
 test1     |                                                            | {}

db1=>
```

- analyst role cannot login
- bob is a member of analyst role
- posgres is a super user
