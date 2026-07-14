
tags: #OpenNet  #data-engineering #data-warehouse   #redshift #mongodb #RDS

---
# Background

Bad or unbounded queries can run for an excessively long time, consuming database resources and impacting the stability of existing production systems. Today there is no enforced upper bound on query execution time across DE's queries.

#### Approach
Add a maximum execution time / timeout to **all queries owned by DE**, set on the query (or session) per database engine. When the limit is exceeded, the database engine **automatically aborts (kills) the query**, protecting shared resources.

**Recommended values:** prefer a _short_ timeout tuned to each query type. Where a tuned value cannot be determined yet, apply a short-term fallback of **1 hour** (3,600,000 ms / 3600s) and revisit later.
#### Acceptance Criteria
- [ ] All DE queries against MySQL set a max execution time (MAX_EXECUTION_TIME hint or session max_execution_time)
- [ ] All DE queries against MongoDB set maxTimeMS on find/aggregate operations
- [ ] All DE queries against PostgreSQL set statement_timeout (session, transaction, or role level)
- [ ] Timeout values are tuned per query type where possible; a 1-hour short-term fallback is applied where a tuned value is not yet determined
- [ ] Verified that a query exceeding the configured limit is automatically aborted by the database engine
#### 