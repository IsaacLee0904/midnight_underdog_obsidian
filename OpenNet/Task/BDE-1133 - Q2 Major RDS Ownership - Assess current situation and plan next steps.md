# Overview
### Background

BI RDS instances have historically been managed by the DBA team, with ad-hoc support from the DA and DE teams when issues arise. As the data infrastructure grows, this arrangement has created gaps in ownership — particularly around proactive monitoring, cost visibility, and scalability planning.

To address this, the DE team is taking formal ownership of these RDS instances going forward.

### Objective

This document aims to：
- Establish a clear inventory of all BI RDS instances (Prod and UAT)
- Assess the current performance and cost of each instance
- Identify existing issues and risks that require immediate or near-term action
- Review the DAGs connected to these instances and understand their dependencies
- Evaluate the current monitoring and alerting coverage
- Deliver a prioritized action plan for the DE team to act on

### Scope


- 任務背景、目的 
- Related Jira or Confluence Link :
	- [Jira Ticket](https://opennetltd.atlassian.net/browse/BDE-1133?atlOrigin=eyJpIjoiYmY0NGE4NDAwNmFkNDBjOGJmZDcxZWZiZjAxNDkzYWYiLCJwIjoiaiJ9)
- Timeline
![[Screenshot 2026-04-30 at 11.05.48 AM.png]]
# Instance Inventory

### Prod Sport

| Alias                     | Endpoint                   | Engine          | Instance Type               | Storage           | Multi-AZ | AWS console                                                                                                                                         |
| :------------------------ | -------------------------- | --------------- | --------------------------- | ----------------- | -------- | --------------------------------------------------------------------------------------------------------------------------------------------------- |
| bi-report-o1 / bigdata-o1 | sporty-pub-prod-bi-main    | Aurora MySQL    | db.r6g.12xlarge             | ~64 TB            | ✅        | [Link](https://eu-central-1.console.aws.amazon.com/rds/home?region=eu-central-1#database:id=sporty-pub-prod-bi-main;is-cluster=true)                |
| bi-bigdata-o1             | sporty-pub-prod-bi-bigdata | MySQL Community | db.r6g.xlarge               | gp3 ( 11,518 GB ) | ✅        | [Link](https://eu-central-1.console.aws.amazon.com/rds/home?region=eu-central-1#database:id=sporty-pub-prod-bi-bigdata-instance-1;is-cluster=false) |
| bigdata-ticket-o1         | bigdata-ticket-prod        | Aurora MySQL    | Serverless v2 (40-100 ACUs) |                   | ❌        | [Link](https://eu-central-1.console.aws.amazon.com/rds/home?region=eu-central-1#database:id=bgd-3kfqd9devzhkx5tb;is-maintenance=true)               |
| metabase-rds              |                            |                 |                             |                   |          |                                                                                                                                                     |
| bet-bi-o1                 | sporty-global-prod-bet-bi  | Aurora MySQL    | erverless v2 (2-40 ACUs)    |                   | ✅        | [Link](https://eu-central-1.console.aws.amazon.com/rds/home?region=eu-central-1#database:id=sporty-global-prod-bet-bi;is-cluster=true)              |


### Prod Encore

### UAT Sport

### UAT Encore

# Performance & Cost

### 每個 instance 的 CPU / Memory / IOPS / connection count

# DAG Review
有哪些 DAG 在寫入或讀取 RDS 的資料？頻率為何？估量每一次的資料量？

# Monitoring Review
