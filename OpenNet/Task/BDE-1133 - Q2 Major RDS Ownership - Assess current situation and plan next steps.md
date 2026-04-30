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

This section documents all BI RDS instances across Sporty and Encore environments (Prod and UAT). For each instance, capture the <font color="#ff0000">engine version</font>, i<font color="#ff0000">nstance type</font>, <font color="#ff0000">storage configuration</font>,  <font color="#ff0000">Multi-AZ setup</font>, and monthly cost. Known issues and observations are noted inline under each sub-section and will be consolidated in Section 6 ( Issues & Risks Summary ). 

### 2.1 Sporty PROD

| Cluster                                                                                                                                                                              | Internal DNS              | Engine                               | Instance Type               | Storage                        | Multi-AZ      | Monthly Cost |
| ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ | ------------------------- | ------------------------------------ | --------------------------- | ------------------------------ | ------------- | ------------ |
| [sporty-pub-prod-bi-main](https://eu-central-1.console.aws.amazon.com/rds/home?region=eu-central-1#database:id=sporty-pub-prod-bi-main;is-cluster=true)                              | bi-report-o1 / bigdata-o1 | Aurora MySQL 8.0.mysql_aurora.3.10.1 | db.r6g.12xlarge             | Aurora Standard (auto-scaling) | Yes (2 Zones) | TBD          |
| [sporty-pub-prod-bi-bigdata-instance-1](https://eu-central-1.console.aws.amazon.com/rds/home?region=eu-central-1#database:id=sporty-pub-prod-bi-bigdata-instance-1;is-cluster=false) | bi-bigdata-o1             | MySQL Community 8.0.40               | db.r6g.xlarge               | gp3 / 11,518 GiB / 12,000 IOPS | Yes           | TBD          |
| [bigdata-ticket-prod](https://eu-central-1.console.aws.amazon.com/rds/home?region=eu-central-1#database:id=bgd-3kfqd9devzhkx5tb;is-maintenance=true)                                 | bigdata-ticket-o1         | Aurora MySQL 3.08.0                  | Serverless v2 (40–100 ACUs) | Aurora Standard (auto-scaling) | No            | TBD          |
| [sporty-global-prod-bet-bi](https://eu-central-1.console.aws.amazon.com/rds/home?region=eu-central-1#database:id=sporty-global-prod-bet-bi;is-cluster=true)                          | bet-bi-o1                 | Aurora MySQL 8.0.mysql_aurora.3.10.1 | Serverless v2 (2–40 ACUs)   | Aurora Standard (auto-scaling) | Yes (2 Zones) | TBD          |

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

| Cluster                                                                                                                                                   | Internal DNS | Engine                               | Instance Type               | Storage                        | Multi-AZ      | Monthly Cost |
| --------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------ | ------------------------------------ | --------------------------- | ------------------------------ | ------------- | ------------ |
| [sporty-pub-uat-bi-main2](https://eu-central-1.console.aws.amazon.com/rds/home?region=eu-central-1#database:id=sporty-pub-uat-bi-main2;is-cluster=true)   | bi-main2-t1  | Aurora MySQL 8.0.mysql_aurora.3.10.1 | db.t4g.medium               | Aurora Standard (auto-scaling) | Yes (2 Zones) | TBD          |
| [sporty-global-uat-bet-bi](https://eu-central-1.console.aws.amazon.com/rds/home?region=eu-central-1#database:id=sporty-global-uat-bet-bi;is-cluster=true) | bet-bi-t1    | Aurora MySQL 8.0.mysql_aurora.3.04.3 | Serverless v2 (0.5–20 ACUs) | Aurora Standard (auto-scaling) | No            | TBD          |

**Notes**
- `sporty-pub-uat-bi-main2`: RDS Extended Support enabled → incurring additional cost
- `sporty-pub-uat-bi-main2`: Enhanced Monitoring disabled → monitoring gap
- `sporty-pub-uat-bi-main2`: Writer CPU at 30.27% on db.t4g.medium (burstable instance) → may hit CPU credit limits under load
- `sporty-global-uat-bet-bi`: Engine version 3.04.3 — significantly behind other instances (3.10.1) → upgrade needed
- `sporty-global-uat-bet-bi`: RDS Extended Support enabled → incurring additional cost
- `sporty-global-uat-bet-bi`: Deletion protection disabled
- `sporty-global-uat-bet-bi`: Enhanced Monitoring disabled → monitoring gap

### 2.3 Encore PROD

| Cluster                                                                                                                                                                       | Internal DNS | Engine                               | Instance Type               | Storage                        | Multi-AZ      | Monthly Cost |
| ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------ | ------------------------------------ | --------------------------- | ------------------------------ | ------------- | ------------ |
| [encore-pub-prod-bi-main-v5-cluster](https://eu-central-1.console.aws.amazon.com/rds/home?region=eu-central-1#database:id=encore-pub-prod-bi-main-v5-cluster;is-cluster=true) | bi-main-o1   | Aurora MySQL 8.0.mysql_aurora.3.10.1 | db.r6g.2xlarge              | Aurora Standard (auto-scaling) | Yes (2 Zones) | TBD          |
| [encore-global-prod-bet-bi](https://eu-central-1.console.aws.amazon.com/rds/home?region=eu-central-1#database:id=encore-global-prod-bet-bi;is-cluster=true)                   | bet-bi-o1    | Aurora MySQL 8.0.mysql_aurora.3.10.1 | Serverless v2 (0.5–20 ACUs) | Aurora Standard (auto-scaling) | Yes (2 Zones) | TBD          |

**Notes**
- `encore-pub-prod-bi-main-v5-cluster`: RDS Extended Support enabled → incurring additional cost
- `encore-pub-prod-bi-main-v5-cluster`: Enhanced Monitoring disabled → monitoring gap
- `encore-global-prod-bet-bi`: RDS Extended Support enabled → incurring additional cost
- `encore-global-prod-bet-bi`: Enhanced Monitoring disabled → monitoring gap

### 2.4 Encore UAT

| Cluster                                                                                                                                                   | Internal DNS | Engine                               | Instance Type               | Storage                        | Multi-AZ | Monthly Cost |
| --------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------ | ------------------------------------ | --------------------------- | ------------------------------ | -------- | ------------ |
| [encore-pub-uat-bi-main](https://eu-central-1.console.aws.amazon.com/rds/home?region=eu-central-1#database:id=encore-pub-uat-bi-main;is-cluster=true)     | bi-main-t1   | Aurora MySQL 8.0.mysql_aurora.3.04.3 | db.t4g.medium               | Aurora Standard (auto-scaling) | No       | TBD          |
| [encore-global-uat-bet-bi](https://eu-central-1.console.aws.amazon.com/rds/home?region=eu-central-1#database:id=encore-global-uat-bet-bi;is-cluster=true) | bet-bi-t1    | Aurora MySQL 8.0.mysql_aurora.3.04.3 | Serverless v2 (0.5–20 ACUs) | Aurora Standard (auto-scaling) | No       | TBD          |

**Notes**
- `encore-pub-uat-bi-main`: Engine version 3.04.3 — significantly behind other instances (3.10.1) → upgrade needed
- `encore-pub-uat-bi-main`: RDS Extended Support enabled → incurring additional cost
- `encore-pub-uat-bi-main`: No Multi-AZ, single Writer only
- `encore-pub-uat-bi-main`: Enhanced Monitoring disabled → monitoring gap
- `encore-global-uat-bet-bi`: Engine version 3.04.3 — significantly behind other instances (3.10.1) → upgrade needed
- `encore-global-uat-bet-bi`: RDS Extended Support enabled → incurring additional cost
- `encore-global-uat-bet-bi`: No Multi-AZ, single Writer only
- `encore-global-uat-bet-bi`: Enhanced Monitoring disabled → monitoring gap

---

# 3. Performance Assessment

TBD — High-level summary to be written after all instance metrics are collected.

### 3.1 Sporty PROD

#### sporty-pub-prod-bi-main

> Cluster has Read / Write separation : instance-1 (Reader) handles queries; instance-2 (Writer) handles all write traffic.

[**instance-1 (Reader)**](https://eu-central-1.console.aws.amazon.com/cloudwatch/home?region=eu-central-1#metricsV2?graph=~(view~'timeSeries~stacked~false~region~'eu-central-1~start~'-PT2160H~end~'P0D)&query=~'*7bAWS*2fRDS*2cDBInstanceIdentifier*7d*20sporty-pub-prod-bi-main-instance-1)

| Metric | Avg | Peak | Risk |
|---|---|---|---|
| CPU Utilization | ~5–10% | ~39.5% | Low |
| DB Connections | ~30–50 | ~105 | Low |
| Freeable Memory | ~120–141 GB | min ~70.73 GB | Medium |
| Read IOPS | ~1–2K | ~15.4K | Low |
| Write IOPS | ~0 | ~0 | — |

[**instance-2 (Writer)**](https://eu-central-1.console.aws.amazon.com/cloudwatch/home?region=eu-central-1#metricsV2?graph=~(view~'timeSeries~stacked~false~region~'eu-central-1~start~'-PT2160H~end~'P0D)&query=~'*7bAWS*2fRDS*2cDBInstanceIdentifier*7d*20sporty-pub-prod-bi-main-instance-2)

| Metric | Avg | Peak | Risk |
|---|---|---|---|
| CPU Utilization | ~20–30% | ~41.7% | Low |
| DB Connections | ~8–16 | ~49 | Low |
| Freeable Memory | ~128–132 GB | min ~114 GB | Low |
| Read IOPS | ~1–2K | ~13.88K | Low |
| Write IOPS | ~15–25K | ~63.81K | Medium |

**Notes**
- Reader memory peaks at ~82% (min freeable ~70.73 GB / 384 GB total) — periodic spikes correlate with pipeline runs
- Reader ReadIOPS spike to 15.4K on 4/23 — investigate which pipeline caused this
- Writer WriteIOPS peak at 63.81K — heavy write load, worth monitoring for sustained spikes

#### sporty-pub-prod-bi-bigdata-instance-1

> MySQL Community instance with Primary + Replica setup. Storage at ~90% capacity.

**instance-1 (Primary)**

| Metric | Avg | Peak | Risk |
|---|---|---|---|
| CPU Utilization | ~20–25% | ~32.8% | Low |
| DB Connections | ~1–2 | ~21 | Low |
| Freeable Memory | ~4.5 GB | min ~4.39 GB | **High** |
| Read IOPS | ~800–1K | ~4.73K | Low |
| Write IOPS | ~3–4K | ~6.25K | Low |

**Notes**
- Memory utilization ~86% (32 GB total, freeable consistently ~4.4 GB) → risk of OOM under additional load
- Very low connection count (avg 1–2) — worth confirming who is connecting to this instance
- Storage at 11,518 / 12,784 GiB (~90% full) → urgent, needs attention

**instance-2 (Replica)**
TBD

#### bigdata-ticket-prod
TBD

#### sporty-global-prod-bet-bi
TBD

### 3.2 Sporty UAT
TBD

### 3.3 Encore PROD
TBD

### 3.4 Encore UAT
TBD

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

