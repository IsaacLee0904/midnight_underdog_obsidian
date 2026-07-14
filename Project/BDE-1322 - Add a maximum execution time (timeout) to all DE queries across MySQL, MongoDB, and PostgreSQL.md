
tags: #OpenNet  #data-engineering #data-warehouse   #redshift #mongodb #RDS

---
## Background

Bad or unbounded queries can run for an excessively long time, consuming database resources and impacting the stability of existing production systems. Today there is no enforced upper bound on query execution time across DE's queries.

To fix this, every DE-owned query gets a maximum execution time, set on the query (or session) per database engine. When the limit is exceeded, the database engine **automatically aborts (kills) the query**, protecting shared resources. Timeout values are tuned per query type where possible; where a tuned value cannot be determined yet, a short-term fallback of **1 hour** (3,600,000 ms) is applied and revisited later.

#### Related Info
ticket : https://opennetltd.atlassian.net/browse/BDE-1322?atlOrigin=eyJpIjoiZDdmNjVkODI5ZDE1NGM5NTgyYzdkZjFhYzZiN2FmYmQiLCJwIjoiaiJ9
Confluence page (per-DAG evidence table) : https://opennetltd.atlassian.net/wiki/spaces/DET/pages/edit-v2/4588011646?draftShareId=d670e855-7ff0-44e0-ab01-919beb903dfb
Related : BI Database Query Hard Limits (Confluence)

---
## Implementation


