
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

DBA can quickly apply global timeout settings to BI clusters, but not to non-BI clusters — so DE enforces timeouts at session / query level in our own code. <mark style="background:#fff88f">Values are centralized in `dags/etl_method/query_timeout.py` (unit = ms, names suffixed `_MS`); each engine applies them through the mechanism that fits it.</mark> (可能會改)

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

### MongoDB

**Mechanism** 
```SQL
db.orders.find({ /* ... */ }).maxTimeMS(3600000)

db.orders.aggregate([ /* ... */ ], { maxTimeMS: 3600000 })
```

**Value**


```SQL
-- for mongo task
WITH ranked AS (
	SELECT
		dag_id,
		task_id,
		duration,
		ROW_NUMBER() OVER (PARTITION BY dag_id, task_id ORDER BY duration) AS rn,
		COUNT(*) OVER (PARTITION BY dag_id, task_id) AS cnt
	FROM airflow.task_instance
	WHERE dag_id IN (
	'sporty_odds.SODerivedMarket',
	'sporty_odds.event_info_history',
	'sporty_odds.event_match_statuses',
	'sporty_odds.golden_source_feed_switch'
	'sporty_odds.manual_settlement_action',
	'sporty_odds.mapped_event',
	'sporty_odds.mapping_history',
	'sporty_odds.monitoring_action_entry',
	'sporty_odds.monitoring_action_entry_backfill',
	'sporty_odds.ui_time_tracking_entry',
	'sporty_odds.uof_message',
	'sporty_odds_beter.event_info_history',
	'sporty_odds_sportradar.event_info_history',
	'encore_rm.order',
	'sporty_rm.order',
	'sporty_rm.ui_time_tracking'
	)
	AND state = 'success'
	AND start_date >= NOW() - INTERVAL 90 DAY
	AND duration IS NOT NULL
)
SELECT
	dag_id,
	task_id,
	MAX(cnt) AS runs,
	ROUND(AVG(duration)) AS avg_sec,
	ROUND(MAX(CASE WHEN rn = CEIL(cnt * 0.95) THEN duration END)) AS p95_sec,
	ROUND(MAX(duration)) AS max_sec
FROM ranked
WHERE task_id not in ("begin_etl", "end_etl")
GROUP BY dag_id, task_id
ORDER BY dag_id, p95_sec DESC;
```
![[Screenshot 2026-07-17 at 4.26.49 PM.png]]


**Known limitation** 

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




