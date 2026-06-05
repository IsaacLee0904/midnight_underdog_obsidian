
tags: #OpenNet  #data-engineering #data-contract #data-quality-check 

---
## Background

As our data platform grows, the number of ETL pipelines and downstream dependencies continues to increase even been used as application. Today, when a DAG is modified — whether it's a transform logic change, a `distkey` adjustment, a column addition, or a data type alteration etc. there is no automated mechanism to notify downstream owners that their tables or dashboards may be affected. Issues are typically only discovered when a consumer notices something unusual, or worse, when it erodes the trust of the BI team.






















### Reference
