
tags: #OpenNet  #data-engineering #data-contract #data-quality-check 

---
## Background

As our data platform grows, the number of ETL pipelines and downstream dependencies continues to increase, some pipelines even serve as application data sources. Today, when a DAG is modified — whether it's a transform logic change, a `distkey` adjustment, a column addition, or a data type alteration — there is no automated mechanism to notify downstream owners. Issues are typically only discovered when a consumer notices something unusual, or worse, when a business stakeholder raises the alarm first, damaging the credibility of the BI team.

To address this, the propose introducing a lightweight **data contract** framework. A data contract is an agreement between data producers and consumers that defines ownership, schema expectations, and quality standards. Rather than adopting a heavyweight implementation, we take an incremental approach — starting with proactive ownership notification at the table level, with the flexibility to evolve toward data quality enforcement and column-level contracts as the system matures.

The good news is that we already have the foundational infrastructure in place. OpenMetadata tracks lineage and ownership across our tables and dashboards. GitHub Actions runs on every PR merge, and our existing CI/CD workflow already leverages Copilot to summarize code changes. The missing piece is connecting these tools into an automated notification loop — so that when a producer modifies a DAG, all downstream owners are alerted before issues propagate. This approach is inspired by [Airbnb's Metis platform](https://medium.com/airbnb-engineering/metis-building-airbnbs-next-generation-data-management-platform-d2c5219edf19), which solves a similar problem at scale by centralizing lineage, ownership, and metadata management across millions of data assets.




















### Reference
* https://soda.io/blog/guide-to-data-contracts
* https://medium.com/airbnb-engineering/metis-building-airbnbs-next-generation-data-management-platform-d2c5219edf19
