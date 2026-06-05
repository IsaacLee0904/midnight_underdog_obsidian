
tags: #OpenNet  #data-engineering #data-contract #data-quality-check 

---
## Background

As our data platform grows, the number of ETL pipelines and downstream dependencies continues to increase — some pipeline are even serve for application data sources. Today, when a DAG is modified — whether it's a transform logic change, a `distkey` adjustment, a column addition, or a data type alteration — there is no automated mechanism to notify downstream owners that their tables or dashboards may be affected. Issues are typically only discovered when a consumer notices something unusual, or worse, when a business stakeholder raises the alarm first — both of which damage the credibility and trust of the BI team.

We already have the foundational infrastructure to solve this problem. OpenMetadata tracks lineage and ownership across our tables and dashboards. GitHub Actions runs on every PR merge in our pipeline repositories, and our existing CI/CD workflow already leverages Copilot to review and summarize code changes. The missing piece is connecting these tools into a lightweight notification loop.

This proposal is inspired by [Airbnb's Metis platform](https://medium.com/airbnb-engineering/metis-building-airbnbs-next-generation-data-management-platform-d2c5219edf19), which centralizes lineage, ownership, and metadata management across millions of data assets. Rather than building a full-scale platform, we aim to achieve similar impact notification goals by leveraging our existing infrastructure — without introducing new tooling.




















### Reference
* https://soda.io/blog/guide-to-data-contracts
* https://medium.com/airbnb-engineering/metis-building-airbnbs-next-generation-data-management-platform-d2c5219edf19
