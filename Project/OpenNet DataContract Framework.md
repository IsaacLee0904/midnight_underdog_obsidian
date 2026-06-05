
tags: #OpenNet  #data-engineering #data-contract #data-quality-check 

---
## Background

As our data platform grows, the number of ETL pipelines and downstream dependencies continues to increase, some pipelines even serve as application data sources. Today, when a DAG is modified — whether it's a transform logic change, a `distkey` adjustment, a column addition, or a data type alteration — there is no automated mechanism to notify downstream owners. Issues are typically only discovered when a consumer notices something unusual, or worse, when a business stakeholder raises the alarm first, damaging the credibility of the BI team.

We already have the foundational infrastructure in place: OpenMetadata tracks lineage and ownership across our tables and dashboards, and GitHub Actions runs on every PR merge. The missing piece is connecting these into an automated notification loop.

This proposal introduces a lightweight **data contract** framework — inspired by [Airbnb's Metis platform](https://medium.com/airbnb-engineering/metis-building-airbnbs-next-generation-data-management-platform-d2c5219edf19). A data contract defines the agreement between data producers and consumers around ownership, schema, and quality expectations. Rather than adopting a heavyweight implementation, we leverage our existing stack: when a DAG change is merged, GitHub Actions calls the OpenMetadata API to identify downstream owners, and automatically sends a Slack notification — ensuring consumers are informed before issues propagate.












## Road Map

- **Phase 1 :** Impact notification for data warehouse DAGs and data analysis DAGs
- **Phase 2 :** Extend coverage upstream to backend application data sources
- **Phase 3 (Nice to have) :** Column-level change detection and data quality enforcement



















### Reference
* https://soda.io/blog/guide-to-data-contracts
* https://medium.com/airbnb-engineering/metis-building-airbnbs-next-generation-data-management-platform-d2c5219edf19
