# The Path of a Query

- [General Flow](#general-flow)
- [Query Optimizer](#query-optimizer)
- [Rule vs Cost Based Optimizer](#rule-vs-cost-based-optimizer)
- [Execution Plan](#execution-plan)

## General Flow

1. Connection - client application establishes a connection to the database server.
2. Parser Stage - check the query for syntax errors and return the abstract syntax tree (AST) of the query.
3. Rewrite System - Adjustments to query that don't affect the result, but can be used to improve performance.
4. Planner/ Optimizer - Evaluate the query and find the best execution plan.
5. Executor - Execute the query according to the execution plan.

## Query Optimizer

1. Generate all possible execution plans

2. Estimate the cost of each plan

- Cost is measured in arbitrary unit
- Using stats from analyzing tables and indexes
- Cost can be used to compare execution plans
- Mostly affected by IO

3. Choose the plan with the lowest cost

- The lower the cost, the faster the execution is

What is cost?

- Measured in arbitrary unit
- Useful for comparing execution plans
- Affected by CPU, IO and other resources
- Highly affected by IO (disk access)

## Rule vs Cost Based Optimizer

1. Rule based optimizer

- Using rules and heuristics to produce an execution plan
- Intuitive and easier to implement

2. Cost based optimizer

- Estimate cost of all possible execution plans using statistics
- Choose the plan with the lowest cost
- Less intuitive and harder to implement
- Expected to produce better plan for most queries

## Execution Plan

```bash
db1=# EXPLAIN SELECT * FROM users;
                      QUERY PLAN
------------------------------------------------------
 Seq Scan on users  (cost=0.00..1.06 rows=6 width=67)
(1 row)

db1=#
```

- View execution plan
- Not execute the query
- Shows estimates

Using Analyze and timing

```bash
db1=# EXPLAIN (ANALYZE ON, TIMING ON) SELECT * FROM users WHERE id > 1;
                                           QUERY PLAN
------------------------------------------------------------------------------------------------
 Seq Scan on users  (cost=0.00..1.07 rows=2 width=67) (actual time=1.522..1.527 rows=7 loops=1)
   Filter: (id > 1)
   Rows Removed by Filter: 1
 Planning Time: 0.129 ms
 Execution Time: 1.554 ms
(5 rows)

db1=#
```
