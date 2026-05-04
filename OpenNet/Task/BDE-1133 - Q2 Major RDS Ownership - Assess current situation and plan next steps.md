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

| Cluster                                                                                                                                                                              | Internal DNS              | Engine                               | Instance Type               | Storage                        | Multi-AZ      |
| ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ | ------------------------- | ------------------------------------ | --------------------------- | ------------------------------ | ------------- |
| [sporty-pub-prod-bi-main](https://eu-central-1.console.aws.amazon.com/rds/home?region=eu-central-1#database:id=sporty-pub-prod-bi-main;is-cluster=true)                              | bi-report-o1 / bigdata-o1 | Aurora MySQL 8.0.mysql_aurora.3.10.1 | db.r6g.12xlarge             | Aurora Standard (auto-scaling) | Yes (2 Zones) |
| [sporty-pub-prod-bi-bigdata-instance-1](https://eu-central-1.console.aws.amazon.com/rds/home?region=eu-central-1#database:id=sporty-pub-prod-bi-bigdata-instance-1;is-cluster=false) | bi-bigdata-o1             | MySQL Community 8.0.40               | db.r6g.xlarge               | gp3 / 11,518 GiB / 12,000 IOPS | Yes           |
| [bigdata-ticket-prod](https://eu-central-1.console.aws.amazon.com/rds/home?region=eu-central-1#database:id=bgd-3kfqd9devzhkx5tb;is-maintenance=true)                                 | bigdata-ticket-o1         | Aurora MySQL 3.08.0                  | Serverless v2 (40–100 ACUs) | Aurora Standard (auto-scaling) | No            |
| [sporty-global-prod-bet-bi](https://eu-central-1.console.aws.amazon.com/rds/home?region=eu-central-1#database:id=sporty-global-prod-bet-bi;is-cluster=true)                          | bet-bi-o1                 | Aurora MySQL 8.0.mysql_aurora.3.10.1 | Serverless v2 (2–40 ACUs)   | Aurora Standard (auto-scaling) | Yes (2 Zones) |

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

| Cluster                                                                                                                                                   | Internal DNS | Engine                               | Instance Type               | Storage                        | Multi-AZ      |
| --------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------ | ------------------------------------ | --------------------------- | ------------------------------ | ------------- |
| [sporty-pub-uat-bi-main2](https://eu-central-1.console.aws.amazon.com/rds/home?region=eu-central-1#database:id=sporty-pub-uat-bi-main2;is-cluster=true)   | bi-main2-t1  | Aurora MySQL 8.0.mysql_aurora.3.10.1 | db.t4g.medium               | Aurora Standard (auto-scaling) | Yes (2 Zones) |
| [sporty-global-uat-bet-bi](https://eu-central-1.console.aws.amazon.com/rds/home?region=eu-central-1#database:id=sporty-global-uat-bet-bi;is-cluster=true) | bet-bi-t1    | Aurora MySQL 8.0.mysql_aurora.3.04.3 | Serverless v2 (0.5–20 ACUs) | Aurora Standard (auto-scaling) | No            |

**Notes**
- `sporty-pub-uat-bi-main2`: RDS Extended Support enabled → incurring additional cost
- `sporty-pub-uat-bi-main2`: Enhanced Monitoring disabled → monitoring gap
- `sporty-pub-uat-bi-main2`: Writer CPU at 30.27% on db.t4g.medium (burstable instance) → may hit CPU credit limits under load
- `sporty-global-uat-bet-bi`: Engine version 3.04.3 — significantly behind other instances (3.10.1) → upgrade needed
- `sporty-global-uat-bet-bi`: RDS Extended Support enabled → incurring additional cost
- `sporty-global-uat-bet-bi`: Deletion protection disabled
- `sporty-global-uat-bet-bi`: Enhanced Monitoring disabled → monitoring gap

### 2.3 Encore PROD

| Cluster                                                                                                                                                                       | Internal DNS | Engine                               | Instance Type               | Storage                        | Multi-AZ      |
| ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------ | ------------------------------------ | --------------------------- | ------------------------------ | ------------- |
| [encore-pub-prod-bi-main-v5-cluster](https://eu-central-1.console.aws.amazon.com/rds/home?region=eu-central-1#database:id=encore-pub-prod-bi-main-v5-cluster;is-cluster=true) | bi-main-o1   | Aurora MySQL 8.0.mysql_aurora.3.10.1 | db.r6g.2xlarge              | Aurora Standard (auto-scaling) | Yes (2 Zones) |
| [encore-global-prod-bet-bi](https://eu-central-1.console.aws.amazon.com/rds/home?region=eu-central-1#database:id=encore-global-prod-bet-bi;is-cluster=true)                   | bet-bi-o1    | Aurora MySQL 8.0.mysql_aurora.3.10.1 | Serverless v2 (0.5–20 ACUs) | Aurora Standard (auto-scaling) | Yes (2 Zones) |

**Notes**
- `encore-pub-prod-bi-main-v5-cluster`: RDS Extended Support enabled → incurring additional cost
- `encore-pub-prod-bi-main-v5-cluster`: Enhanced Monitoring disabled → monitoring gap
- `encore-global-prod-bet-bi`: RDS Extended Support enabled → incurring additional cost
- `encore-global-prod-bet-bi`: Enhanced Monitoring disabled → monitoring gap

### 2.4 Encore UAT

| Cluster                                                                                                                                                   | Internal DNS | Engine                               | Instance Type               | Storage                        | Multi-AZ |
| --------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------ | ------------------------------------ | --------------------------- | ------------------------------ | -------- |
| [encore-pub-uat-bi-main](https://eu-central-1.console.aws.amazon.com/rds/home?region=eu-central-1#database:id=encore-pub-uat-bi-main;is-cluster=true)     | bi-main-t1   | Aurora MySQL 8.0.mysql_aurora.3.04.3 | db.t4g.medium               | Aurora Standard (auto-scaling) | No       |
| [encore-global-uat-bet-bi](https://eu-central-1.console.aws.amazon.com/rds/home?region=eu-central-1#database:id=encore-global-uat-bet-bi;is-cluster=true) | bet-bi-t1    | Aurora MySQL 8.0.mysql_aurora.3.04.3 | Serverless v2 (0.5–20 ACUs) | Aurora Standard (auto-scaling) | No       |

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

# 3. Performance & Cost Assessment

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

[**instance-1 (Primary)**](https://eu-central-1.console.aws.amazon.com/cloudwatch/home?region=eu-central-1#metricsV2?graph=~(view~'timeSeries~stacked~false~region~'eu-central-1~start~'-PT2160H~end~'P0D)&query=~'*7bAWS*2fRDS*2cDBInstanceIdentifier*7d*20sporty-pub-prod-bi-bigdata-instance-1)

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

[**instance-2 (Replica)**](https://eu-central-1.console.aws.amazon.com/cloudwatch/home?region=eu-central-1#metricsV2?graph=~(view~'timeSeries~stacked~false~region~'eu-central-1~start~'-PT2160H~end~'P0D)&query=~'*7bAWS*2fRDS*2cDBInstanceIdentifier*7d*20sporty-pub-prod-bi-bigdata-instance-2)

| Metric | Avg | Peak | Risk |
|---|---|---|---|
| CPU Utilization | ~2–4% | ~18.9% | Low |
| DB Connections | ~0 | ~2 | — |
| Freeable Memory | ~4.2–4.4 GB (↓ trending) | min ~4.19 GB | **High** |
| Read IOPS | ~0 | ~525 | Low |
| Write IOPS | ~300–600 | ~3.78K | Low |

**Notes**
- Replica has near-zero connections and read IOPS — appears to be unused for actual read traffic
- Freeable Memory on a clear downward trend (4.41 GB → 4.19 GB over 2 weeks) → potential slow memory leak, risk of OOM if not addressed
- Worth investigating whether this Replica is still needed

#### bigdata-ticket-prod

> Serverless v2 (40–100 ACUs), currently under Blue/Green migration to provisioned r8g.8xlarge (DBA-7596). Metrics below reflect Blue (current production) side.

[**instance-1 (Writer)**](https://eu-central-1.console.aws.amazon.com/cloudwatch/home?region=eu-central-1#metricsV2?graph=~(view~'timeSeries~stacked~false~region~'eu-central-1~start~'-PT2160H~end~'P0D~stat~'Average~period~60)&query=~'*7bAWS*2fRDS*2cDBInstanceIdentifier*7d*20*20sporty-global-prod-bet-bi-instance-1)

| Metric          | Avg        | Peak          | Risk     |
| --------------- | ---------- | ------------- | -------- |
| CPU Utilization | ~40–60%    | ~82.8%        | **High** |
| DB Connections  | ~40–80     | ~117          | Medium   |
| Freeable Memory | ~88–132 GB | min ~44.85 GB | Medium   |
| Read IOPS       | ~300–800   | ~15.29K       | Low      |
| Write IOPS      | ~10–20K    | ~32.79K       | **High** |

**Notes**
- CPU avg 40–60% with peaks at 82.8% — likely approaching Serverless 100 ACU ceiling, explains recurring instability
- Write IOPS consistently 10–20K is the primary driver of HLL (History List Length) issues
- ReadIOPS spike to 15.29K around 4/29–30 likely related to Blue/Green switchover activities
- Memory deep dips (~44.85 GB freeable) correlate with high write load periods
- Migration to provisioned r8g.8xlarge in progress — expected to improve stability and cost

[**instance-2 (Reader)**](https://eu-central-1.console.aws.amazon.com/cloudwatch/home?region=eu-central-1#metricsV2?graph=~(view~'timeSeries~stacked~false~region~'eu-central-1~start~'-PT2160H~end~'P0D~stat~'Average~period~60)&query=~'*7bAWS*2fRDS*2cDBInstanceIdentifier*7d*20bigdata-ticket-prod-instance-2)

| Metric | Avg | Peak | Risk |
|---|---|---|---|
| CPU Utilization | ~5–15% | ~60.2% | Medium |
| DB Connections | N/A | N/A | — |
| Freeable Memory | ~88–136 GB | min ~23.86 GB | **High** |
| Read IOPS | ~0–2K (bursty) | ~16.54K | Medium |
| Write IOPS | ~0 | ~0 | — |

**Notes**
- Memory dips to ~23.86 GB freeable during read bursts — more severe than Writer
- Read pattern is highly bursty (spikes to 16.54K then drops to 0) — correlates with batch pipeline reads
- CPU peaks at 60.2% during heavy read periods

#### sporty-global-prod-bet-bi

> Aurora Serverless v2 (2–40 ACUs) cluster with Reader + Writer separation.

[**instance-1 (Reader)**](https://eu-central-1.console.aws.amazon.com/cloudwatch/home?region=eu-central-1#metricsV2?graph=~(view~'timeSeries~stacked~false~region~'eu-central-1~start~'-PT2160H~end~'P0D~stat~'Average~period~60)&query=~'*7bAWS*2fRDS*2cDBInstanceIdentifier*7d*20sporty-global-prod-bet-bi-instance-1)

| Metric | Avg | Peak | Risk |
|---|---|---|---|
| CPU Utilization | ~2–4% | ~8.99% | Low |
| DB Connections | ~600–1,000 | ~1.44K | Medium |
| Freeable Memory | ~70–75 GB | min ~33.43 GB | Medium |
| Read IOPS | ~50–200 | ~742 | Low |
| Write IOPS | ~0 | ~0 | — |

**Notes**
- Connection count (~600–1,000 avg) is notably high for a Serverless 2–40 ACU instance — worth investigating which services are connecting
- Memory dips to ~33.43 GB freeable periodically — correlates with connection/read spikes

[**instance-2 (Writer)**](https://eu-central-1.console.aws.amazon.com/cloudwatch/home?region=eu-central-1#metricsV2?graph=~(view~'timeSeries~stacked~false~region~'eu-central-1~start~'-PT2160H~end~'P0D~stat~'Average~period~60)&query=~'*7bAWS*2fRDS*2cDBInstanceIdentifier*7d*20sporty-global-prod-bet-bi-instance-2)

| Metric | Avg | Peak | Risk |
|---|---|---|---|
| CPU Utilization | ~0–5% (daily spike ~32–40%) | ~63% | Medium |
| DB Connections | ~0–1 | ~4 | Low |
| Freeable Memory | ~70–77 GB | min ~33.32 GB | Medium |
| Read IOPS | ~400–800 (during batch) | ~1.65K | Low |
| Write IOPS | ~0 (daily spike ~1.8–2.1K) | ~3.61K | Low |

**Notes**
- Clear daily spike pattern on CPU/WriteIOPS — consistent with a scheduled batch pipeline running once per day
- DB Connections extremely low (avg 0–1) — batch-only access, no interactive queries on Writer
- Anomalous event on 4/20: CPU peaked at 63%, Memory dipped to ~33 GB, WriteIOPS hit 3.61K — investigate whether a non-routine pipeline ran that day
- Strong contrast between instance-1 Reader (~600–1,000 connections) and instance-2 Writer (~0) — need to identify what is holding persistent connections to the Reader

### 3.2 Sporty UAT

#### sporty-pub-uat-bi-main2

> Aurora cluster with Writer + Reader separation. db.t4g.medium (burstable, 4 GB RAM). Metrics show two distinct phases: active (4/19–4/22) and near-idle (4/23 onwards).

[**instance-1 (Writer)**](https://eu-central-1.console.aws.amazon.com/cloudwatch/home?region=eu-central-1#metricsV2?graph=~(view~'timeSeries~stacked~false~region~'eu-central-1~start~'-PT2160H~end~'P0D~stat~'Average~period~60)&query=~'*7bAWS*2fRDS*2cDBInstanceIdentifier*7d*20sporty-pub-uat-bi-main2-instance-1)

| Metric | Avg | Peak | Risk |
|---|---|---|---|
| CPU Utilization | ~25–30% (active) / ~10–13% (post 4/22) | ~64.9% | Medium |
| DB Connections | ~6–9 (active) / ~0 (post 4/22) | ~18 | Low |
| Freeable Memory | ~741 MB–1.1 GB (active) / ~1.35 GB (post 4/22) | min ~741 MB | **High** |
| Read IOPS | ~0 (between spikes) | ~3.51K | Low |
| Write IOPS | ~40–70 (active) / ~0 (post 4/22) | ~131.92 | Low |

[**instance-2 (Reader)**](https://eu-central-1.console.aws.amazon.com/cloudwatch/home?region=eu-central-1#metricsV2?graph=~(view~'timeSeries~stacked~false~region~'eu-central-1~start~'-PT2160H~end~'P0D~stat~'Average~period~60)&query=~'*7bAWS*2fRDS*2cDBInstanceIdentifier*7d*20sporty-pub-uat-bi-main2-instance-2)

| Metric | Avg | Peak | Risk |
|---|---|---|---|
| CPU Utilization | ~20–30% | ~64% | Medium |
| DB Connections | ~5–8 | ~21 | Low |
| Freeable Memory | ~880 MB–1.08 GB (↑ trending up) | min ~779 MB | **High** |
| Read IOPS | ~0 (daily spike) | ~4.11K | Low |
| Write IOPS | ~20–50 (replication) | ~435.22 | Low |

**Notes**
- Memory utilization ~80% (min freeable ~779 MB on 4 GB total) — high for a UAT instance
- Daily ReadIOPS spikes (~4.11K) indicate a batch read pipeline routing reads to this Reader consistently
- Anomalous WriteIOPS spike on 4/28 (~435.22) — significantly above baseline replication writes, needs investigation
- Notable contrast: instance-1 (Writer) went near-idle after 4/22 while this Reader remains consistently active — UAT writes have stopped but something continues reading from this cluster

#### sporty-global-uat-bet-bi

> Aurora Serverless v2 (0.5–20 ACUs), single instance (no Reader/Writer split).

[**instance-1 (Writer)**](https://eu-central-1.console.aws.amazon.com/cloudwatch/home?region=eu-central-1#metricsV2?graph=~(view~'timeSeries~stacked~false~region~'eu-central-1~start~'-PT2160H~end~'P0D~stat~'Average~period~60)&query=~'*7bAWS*2fRDS*2cDBInstanceIdentifier*7d*20sporty-global-uat-bet-bi-instance-1)

| Metric | Avg | Peak | Risk |
|---|---|---|---|
| CPU Utilization | ~2.9% | ~4.63% | Low |
| DB Connections | ~140–170 | ~183 | **High** |
| Freeable Memory | ~36–38 GB | min ~30.18 GB | Low |
| Read IOPS | ~0 | ~0.03 | — |
| Write IOPS | ~3.79 | ~8.4 | Low |

**Notes**
- DB Connections (~140–170) is unexpectedly high for a UAT Serverless instance with near-zero read/write activity — likely idle connections held by a connection pool that is not releasing properly
- ReadIOPS essentially zero despite high connection count — connections are sleeping/idle, no actual queries being executed
- Memory dips correlate with connection spikes but remain at acceptable levels overall

### 3.3 Encore PROD

#### encore-pub-prod-bi-main-v5-cluster

> Aurora cluster (db.r6g.2xlarge, 64 GB RAM) with Reader + Writer separation.

[**encore-pub-prod-bi-main-v5 (Reader)**](https://eu-central-1.console.aws.amazon.com/cloudwatch/home?region=eu-central-1#metricsV2?graph=~(view~'timeSeries~stacked~false~region~'eu-central-1~start~'-PT2160H~end~'P0D~stat~'Average~period~60)&query=~'*7bAWS*2fRDS*2cDBInstanceIdentifier*7d*20*20encore-pub-prod-bi-main-v5)

| Metric | Avg | Peak | Risk |
|---|---|---|---|
| CPU Utilization | ~25–40% | ~71% | **High** |
| DB Connections | ~8–17 | ~51 | Low |
| Freeable Memory | ~29 GB | min ~25.59 GB | Low |
| Read IOPS | ~4–6K | ~10.54K | Medium |
| Write IOPS | ~0 | ~0 | — |

[**encore-pub-prod-bi-main-v5-instance-2 (Writer)**](https://eu-central-1.console.aws.amazon.com/cloudwatch/home?region=eu-central-1#metricsV2?graph=~(view~'timeSeries~stacked~false~region~'eu-central-1~start~'-PT2160H~end~'P0D~stat~'Average~period~60)&query=~'*7bAWS*2fRDS*2cDBInstanceIdentifier*7d*20encore-pub-prod-bi-main-v5-instance-2)

| Metric | Avg | Peak | Risk |
|---|---|---|---|
| CPU Utilization | ~30–50% | ~82.3% | **High** |
| DB Connections | ~4–8 | ~25 | Low |
| Freeable Memory | ~25.87–26.67 GB | min ~23.39 GB | Low |
| Read IOPS | ~200–500 | ~2.86K | Low |
| Write IOPS | ~2–4K | ~11.23K | **High** |

**Notes**
- Writer CPU consistently at 30–50% with peak at 82.3% — concerning for a db.r6g.2xlarge; CPU trend appears to be increasing from 5/02 onwards
- Writer WriteIOPS sustained at 2–4K around the clock — heavy continuous write load, peak 11.23K
- Reader CPU peaks at 71% with sustained ReadIOPS 4–6K — both instances under significant pressure
- Cluster-wide concern: both instances are heavily utilized; may need instance type upgrade (consider db.r6g.4xlarge or larger)

#### encore-global-prod-bet-bi

> Aurora Serverless v2 (0.5–20 ACUs) cluster with Reader + Writer separation.

**instance-1 (Reader)**

| Metric | Avg | Peak | Risk |
|---|---|---|---|
| CPU Utilization | ~2.21% | ~5.8% | Low |
| DB Connections | ~38–40 (baseline) | ~90 | Medium |
| Freeable Memory | ~39 GB | min ~28.29 GB | Low |
| Read IOPS | ~0 | ~8.76 | — |
| Write IOPS | ~0 | ~0 | — |

**instance-2 (Writer)**

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

