# 1. Overview
### 1.1 Background

BI RDS instances have historically been managed by the DBA team, with ad-hoc support from the DA and DE teams when issues arise. As the data infrastructure grows, this arrangement has created gaps in ownership — particularly around proactive monitoring, cost visibility, and scalability planning.

To address this, the [DE team is taking formal ownership of these RDS instances](https://opennetltd.atlassian.net/browse/BDE-1133?atlOrigin=eyJpIjoiZTcxMTJjMjliYmFhNDIyNGE1Y2RiMzljOWRmMGU3YzciLCJwIjoiaiJ9) going forward.

### 1.2 Objective

This document aims to：
- Establish a clear inventory of all BI RDS instances (Prod and UAT)
- Assess the current performance and cost of each instance
- Identify existing issues and risks that require immediate or near-term action
- Review the DAGs connected to these instances and understand their dependencies
- Evaluate the current monitoring and alerting coverage
- Deliver a prioritized action plan for the DE team to act on

### 1.3 Scope

This assessment covers BI RDS instances only, excluding service backends such as Airflow and Metabase RDS.

![[Screenshot 2026-04-30 at 11.15.56 AM.png]]

---

# 2. Instance Inventory

### 2.1 Sporty PROD

| Cluster                               | Internal DNS              | Engine                               | Instance Type   | Storage                        | Multi-AZ      | Monthly Cost |
| ------------------------------------- | ------------------------- | ------------------------------------ | --------------- | ------------------------------ | ------------- | ------------ |
| sporty-pub-prod-bi-main               | bi-report-o1 / bigdata-o1 | Aurora MySQL 8.0.mysql_aurora.3.10.1 | db.r6g.12xlarge | Aurora Standard (auto-scaling) | Yes (2 Zones) | TBD          |
| sporty-pub-prod-bi-bigdata-instance-1 | bi-bigdata-o1             | MySQL Community 8.0.40               | db.r6g.xlarge   | gp3 / 11,518 GiB / 12,000 IOPS | Yes           | TBD          |
| bigdata-ticket-prod                   | bigdata-ticket-o1         | Aurora MySQL 3.08.0                  | Serverless v2 (40–100 ACUs) | Aurora Standard (auto-scaling) | No | TBD |
| sporty-global-prod-bet-bi             | bet-bi-o1                 | Aurora MySQL 8.0.mysql_aurora.3.10.1 | Serverless v2 (2–40 ACUs) | Aurora Standard (auto-scaling) | Yes (2 Zones) | TBD |

**Notes**
- `sporty-pub-prod-bi-main`: RDS Extended Support enabled → incurring additional cost
- `sporty-pub-prod-bi-bigdata-instance-1`: Storage at 11,518 / 12,784 GiB (~90% full) → risk of hitting limit
- `sporty-pub-prod-bi-bigdata-instance-1`: Performance Insights disabled → monitoring gap
- `sporty-pub-prod-bi-bigdata-instance-1`: Enhanced Monitoring disabled → monitoring gap
- `bigdata-ticket-prod`: Migration from Serverless to provisioned (r8g.8xlarge) in progress — DBA-7596
- `bigdata-ticket-prod`: Recurring HLL (History List Length) issues — root cause under investigation
- `sporty-global-prod-bet-bi`: RDS Extended Support enabled → incurring additional cost
- `sporty-global-prod-bet-bi`: Enhanced Monitoring disabled → monitoring gap


### 2.2 Sporty UAT

| Cluster                  | Internal DNS | Engine                               | Instance Type               | Storage                        | Multi-AZ      | Monthly Cost |
| ------------------------ | ------------ | ------------------------------------ | --------------------------- | ------------------------------ | ------------- | ------------ |
| sporty-pub-uat-bi-main2  | bi-main2-t1  | Aurora MySQL 8.0.mysql_aurora.3.10.1 | db.t4g.medium               | Aurora Standard (auto-scaling) | Yes (2 Zones) | TBD          |
| sporty-global-uat-bet-bi | bet-bi-t1    | Aurora MySQL 8.0.mysql_aurora.3.04.3 | Serverless v2 (0.5–20 ACUs) | Aurora Standard (auto-scaling) | No            | TBD          |

**Notes**
- `sporty-pub-uat-bi-main2`: RDS Extended Support enabled → incurring additional cost
- `sporty-pub-uat-bi-main2`: Enhanced Monitoring disabled → monitoring gap
- `sporty-pub-uat-bi-main2`: Writer CPU at 30.27% on db.t4g.medium (burstable instance) → may hit CPU credit limits under load
- `sporty-global-uat-bet-bi`: Engine version 3.04.3 — significantly behind other instances (3.10.1) → upgrade needed
- `sporty-global-uat-bet-bi`: RDS Extended Support enabled → incurring additional cost
- `sporty-global-uat-bet-bi`: Deletion protection disabled
- `sporty-global-uat-bet-bi`: Enhanced Monitoring disabled → monitoring gap

### 2.3 Encore PROD

| Cluster | Internal DNS | Engine | Instance Type | Storage | Multi-AZ | Monthly Cost |
|---|---|---|---|---|---|---|
| encore-pub-prod-bi-main-v5-cluster | bi-main-o1 | MySQL | TBD | TBD | TBD | TBD |
| encore-global-prod-bet-bi | bet-bi-o1 | MySQL | TBD | TBD | TBD | TBD |

### 2.4 Encore UAT

| Cluster | Internal DNS | Engine | Instance Type | Storage | Multi-AZ | Monthly Cost |
|---|---|---|---|---|---|---|
| encore-pub-uat-bi-main | bi-main-t1 | MySQL | TBD | TBD | TBD | TBD |
| encore-global-uat-bet-bi | bet-bi-t1 | MySQL | TBD | TBD | TBD | TBD |

---

### Backlog
```txt
1. Overview
   - 背景與目標

2. Instance Inventory
   - 2.1 Prod Sporty
   - 2.2 Prod Encore
   - 2.3 UAT Sporty
   - 2.4 UAT Encore
   （每個 instance 用相同欄位模板）

3. Performance & Cost Assessment
   - 3.1 CPU / Memory / IOPS / Connections
   - 3.2 Monthly Cost (Cost Explorer)
   - 3.3 Benchmark vs. Threshold

4. DAG Review
   - 4.1 DAG ↔ Instance Dependency Map
   - 4.2 Read/Write Patterns & Schedules
   - 4.3 Known Issues

5. Monitoring Review
   - 5.1 CloudWatch Alarm Coverage
   - 5.2 Alert Notification Setup
   - 5.3 Gaps & Blind Spots

6. Issues & Risks Summary
   （集中整理所有發現的問題）

7. Action Plan
   - P0 / P1 / P2 分級
   - Owner & Timeline
```

