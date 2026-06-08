
tags: #OpenNet  #data-engineering #data-contract #data-quality-check 

---
## Background

As our data platform grows, the number of ETL pipelines and downstream dependencies continues to increase, some pipelines even serve as application data sources. Today, when a DAG / table is modified — whether it's a transform logic change, a `distkey` adjustment, a column addition, or a data type alteration — there is no automated mechanism to notify downstream owners. Issues are typically only discovered when a consumer notices something unusual, or worse, when a business stakeholder raises the alarm first, damaging the credibility of the BI team.

This proposal introduces a lightweight **data contract** framework, inspired by [Airbnb's Metis platform](https://medium.com/airbnb-engineering/metis-building-airbnbs-next-generation-data-management-platform-d2c5219edf19), to address this gap. By ensuring every object (table & dashboard) has a registered owner in OpenMetadata, we can trigger an automated workflow to notify downstream owners whenever an upstream table is modified.

## Data Contract

The proposal architecture :

![[Datacontract.excalidraw|1500]]




Repo needed to include
* DAGs in [warehouse_engineer](https://github.com/opennetltd/warehouse_engineer)
* DAGs in [data_analysis](https://github.com/opennetltd/data_analysis)
* DDL statement in [dba-redshift-executor-prod](https://github.com/opennetltd/dba-redshift-executor-prod)









## Open Questions

- Slack notification target: DM to downstream owners vs. dedicated alert channel (which channel)?
- Notification content: PR link, affected table, downstream impact scope?
- Ownership coverage : How to using OpenMetadata to coverage all the owner and how do we handle tables without a registered owner (edge case)?

## Road Map

- **Phase 1 :** Impact notification for data warehouse DAGs and data analysis DAGs
- **Phase 2 :** Extend coverage upstream to backend application data sources
- **Phase 3 (Nice to have) :** Column-level change detection and data quality enforcement



















### Reference
* https://soda.io/blog/guide-to-data-contracts
* https://medium.com/airbnb-engineering/metis-building-airbnbs-next-generation-data-management-platform-d2c5219edf19
