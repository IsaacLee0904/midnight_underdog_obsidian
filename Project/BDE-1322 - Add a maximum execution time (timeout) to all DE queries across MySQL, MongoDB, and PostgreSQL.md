
tags: #OpenNet  #data-engineering #data-warehouse   #redshift #mongodb #RDS

---
## Background

Bad or unbounded queries can run for an excessively long time, consuming database resources and impacting the stability of existing production systems. Today there is no enforced upper bound on query execution time across DE's queries.

To fix this, every DE-owned query gets a maximum execution time, set on the query (or session) per database engine. When the limit is exceeded, the database engine **automatically aborts (kills) the query**, protecting shared resources. Timeout values are tuned per query type where possible; where a tuned value cannot be determined yet, a short-term fallback of **1 hour** (3,600,000 ms) is applied and revisited later.

Scope :
- [ ] All DE queries against MySQL set a max execution time (`MAX_EXECUTION_TIME` hint or session `max_execution_time`)
- [ ] All DE queries against MongoDB set `maxTimeMS` on find/aggregate operations
- [ ] All DE queries against PostgreSQL set `statement_timeout` (session, transaction, or role level)
- [ ] Timeout values are tuned per query type where possible; a 1-hour short-term fallback is applied where a tuned value is not yet determined
- [ ] Verified that a query exceeding the configured limit is automatically aborted by the database engine

Related Info : 
* ticket : https://opennetltd.atlassian.net/browse/BDE-1322?atlOrigin=eyJpIjoiZDdmNjVkODI5ZDE1NGM5NTgyYzdkZjFhYzZiN2FmYmQiLCJwIjoiaiJ9

---
## Implementation

Design premise : DBA can quickly apply global timeout settings to BI clusters, but not to non-BI clusters — so DE enforces timeouts at session / query level in our own code. <mark style="background:#fff88f">Values are centralized in `dags/etl_method/query_timeout.py` (unit = ms, names suffixed `_MS`); each engine applies them through the mechanism that fits it.</mark> (可能會改)

### MySQL

**Mechanism** 
```SQL
-- Optimizer hint (per statement)
SELECT /*+ MAX_EXECUTION_TIME(3600000) */ col FROM orders WHERE ...;

-- Or per session
SET SESSION max_execution_time = 3600000;  -- 1 hour
```

session-level `max_execution_time`, set inside the shared query functions (`run_sql_in_mysql` / `mysql_sql_to_dataframe`) — one choke point covers all callers. Helper `_set_mysql_session_max_execution_time` by Kevin Wei; default 10 min, per-call override via `max_execution_time_min`, `0` to disable, connections in the `mysql_max_execution_time_excluded_conns` Variable are skipped.

**Known limitation** : `max_execution_time` only bounds top-level read-only SELECT. `INSERT ... SELECT` / CTAS / UPDATE / DELETE / `SELECT INTO OUTFILE S3` are NOT bounded → writes to BI mart DBs remain unprotected (raised to DBA separately).

**Timeout values** : (TBD — tune with 90-day task duration stats, same method as MongoDB)

**Changes** : (TBD — Kevin's PR, repo/link)

### MongoDB

**Mechanism** 
```SQL
db.orders.find({ /* ... */ }).maxTimeMS(3600000)

db.orders.aggregate([ /* ... */ ], { maxTimeMS: 3600000 })
```

### PostgreSQL

**Mechanism** 
```SQL
-- Per session
SET statement_timeout = '3600s';   -- or 3600000

-- Per transaction
SET LOCAL statement_timeout = 3600000;

-- Per role (optional, applies to all that role's connections)
ALTER ROLE de_app SET statement_timeout = '3600s';
```


---
## Verification & Impact

- [ ] Kill verification : run a query with `maxTimeMS=1000` against a large collection in the test DAG (`test_mongodb_connection`), expect pymongo `ExecutionTimeout` — attach evidence to ticket
- Impact : (TBD — what is protected after rollout, remaining gaps)


