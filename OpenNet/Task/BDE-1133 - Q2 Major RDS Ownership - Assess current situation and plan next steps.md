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

### 1.4 Instance History

The current BI RDS footprint evolved over multiple phases:

1. **bi-report-o1** : The original BI database, functioning as a Redshift data warehouse currently. Data was synchronized from service-side RDS instances via cron jobs.
2. **bigdata-ticket-o1** : Added to serve serverless use cases (15-minute sync cadence), primarily used by the Trading team.
3. **Airflow migration** : Most pipeline work migrated from cron jobs to Airflow, with DAGs primarily maintained in the [data_analysis](https://github.com/opennetltd/data_analysis) repository.
4. **bi-bigdata-o1** : Created to offload workload from bi-report-o1 as data volume grew.

Other environments (Encore, UAT) followed a similar evolution. As of this assessment, the majority of pipeline work has been migrated to Airflow.

**Current Data Flow**

The primary data flow today is:

```
Service RDS → Redshift → BI RDS → Metabase (dashboard)
```

A secondary flow still exists for certain use cases:

```
Service RDS → BI RDS (direct)
```

---

# 2. Instance Inventory

This section documents all BI RDS instances across Sporty and Encore environments (Prod and UAT). For each instance, capture the <font color="#ff0000">engine version</font>, <font color="#ff0000">instance type</font>, <font color="#ff0000">storage configuration</font>, <font color="#ff0000">Multi-AZ setup</font>, and monthly cost. Known issues and observations are noted inline under each sub-section and will be consolidated in Section 6 ( Issues & Risks Summary ). 

### 2.1 Sporty PROD

#### [sporty-pub-prod-bi-main](https://eu-central-1.console.aws.amazon.com/rds/home?region=eu-central-1#database:id=sporty-pub-prod-bi-main;is-cluster=true)

Cluster has Read / Write separation sporty-pub-prod-bi-main-instance-1 handles queries; instance-2 (Writer) handles all write traffic. And used to serve as data warehouse (like the Redshift now). Provided for DA write data directorly from service RDS our from Redshift and query by metabase

1. **Endpoint** : 
	*  bi-report-o1.mysql.pub.s.sportybet
	*  bigdata-o1.mysql.pub.s.sportybet
2. Engine：Aurora MySQL 8.0.mysql_aurora.3.10.1
3. Instance Type：db.r6g.12xlarge

[**sporty-pub-prod-bi-main-instance-1 (Reader)**](https://eu-central-1.console.aws.amazon.com/cloudwatch/home?region=eu-central-1#metricsV2?graph=~(view~'timeSeries~stacked~false~region~'eu-central-1~start~'-PT2160H~end~'P0D)&query=~'*7bAWS*2fRDS*2cDBInstanceIdentifier*7d*20sporty-pub-prod-bi-main-instance-1)

| Metric          | Avg         | Peak          | Risk   |
| --------------- | ----------- | ------------- | ------ |
| CPU Utilization | ~5–10%      | ~39.5%        | Low    |
| DB Connections  | ~30–50      | ~105          | Low    |
| Freeable Memory | ~120–141 GB | min ~70.73 GB | Medium |
| Read IOPS       | ~1–2K       | ~15.4K        | Low    |
| Write IOPS      | ~0          | ~0            | —      |

[**sporty-pub-prod-bi-main-instance-2 (Writer)**](https://eu-central-1.console.aws.amazon.com/cloudwatch/home?region=eu-central-1#metricsV2?graph=~(view~'timeSeries~stacked~false~region~'eu-central-1~start~'-PT2160H~end~'P0D)&query=~'*7bAWS*2fRDS*2cDBInstanceIdentifier*7d*20sporty-pub-prod-bi-main-instance-2)

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
1. Storage：Aurora Standard (auto-scaling)
2. Multi-AZ：Yes

#### [sporty-pub-prod-bi-bigdata-instance-1](https://eu-central-1.console.aws.amazon.com/rds/home?region=eu-central-1#database:id=sporty-pub-prod-bi-bigdata-instance-1;is-cluster=false)




| Cluster                                                                                                                                              | Endpoint                  | Engine                 | Instance Type               | Storage                        | Multi-AZ      |
| ---------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------- | ---------------------- | --------------------------- | ------------------------------ | ------------- |
|                                                                                                                                                      | bi-report-o1 / bigdata-o1 |                        | db.r6g.12xlarge             | Aurora Standard (auto-scaling) | Yes (2 Zones) |
|                                                                                                                                                      | bi-bigdata-o1             | MySQL Community 8.0.40 | db.r6g.xlarge               | gp3 / 11,518 GiB / 12,000 IOPS | Yes           |
| [bigdata-ticket-prod](https://eu-central-1.console.aws.amazon.com/rds/home?region=eu-central-1#database:id=bgd-3kfqd9devzhkx5tb;is-maintenance=true) | bigdata-ticket-o1         | Aurora MySQL 3.08.0    | Serverless v2 (40–100 ACUs) | Aurora Standard (auto-scaling) | No            |

**Notes**
- `sporty-pub-prod-bi-main`: RDS Extended Support enabled → incurring additional cost
- `sporty-pub-prod-bi-bigdata-instance-1`: Storage at 11,518 / 12,784 GiB (~90% full) → risk of hitting limit
- `sporty-pub-prod-bi-bigdata-instance-1`: Performance Insights disabled → monitoring gap
- `sporty-pub-prod-bi-bigdata-instance-1`: Enhanced Monitoring disabled → monitoring gap
- `bigdata-ticket-prod`: Migration from Serverless to provisioned (r8g.8xlarge) in progress — DBA-7596
- `bigdata-ticket-prod`: Recurring HLL (History List Length) issues — root cause under investigation

### 2.2 Sporty UAT

| Cluster                                                                                                                                                   | Endpoint | Engine                               | Instance Type               | Storage                        | Multi-AZ      |
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

| Cluster                                                                                                                                                                       | Endpoint | Engine                               | Instance Type               | Storage                        | Multi-AZ      |
| ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------ | ------------------------------------ | --------------------------- | ------------------------------ | ------------- |
| [encore-pub-prod-bi-main-v5-cluster](https://eu-central-1.console.aws.amazon.com/rds/home?region=eu-central-1#database:id=encore-pub-prod-bi-main-v5-cluster;is-cluster=true) | bi-main-o1   | Aurora MySQL 8.0.mysql_aurora.3.10.1 | db.r6g.2xlarge              | Aurora Standard (auto-scaling) | Yes (2 Zones) |
| [encore-global-prod-bet-bi](https://eu-central-1.console.aws.amazon.com/rds/home?region=eu-central-1#database:id=encore-global-prod-bet-bi;is-cluster=true)                   | bet-bi-o1    | Aurora MySQL 8.0.mysql_aurora.3.10.1 | Serverless v2 (0.5–20 ACUs) | Aurora Standard (auto-scaling) | Yes (2 Zones) |

**Notes**
- `encore-pub-prod-bi-main-v5-cluster`: RDS Extended Support enabled → incurring additional cost
- `encore-pub-prod-bi-main-v5-cluster`: Enhanced Monitoring disabled → monitoring gap
- `encore-global-prod-bet-bi`: RDS Extended Support enabled → incurring additional cost
- `encore-global-prod-bet-bi`: Enhanced Monitoring disabled → monitoring gap

### 2.4 Encore UAT

| Cluster                                                                                                                                                   | Endpoint | Engine                               | Instance Type               | Storage                        | Multi-AZ |
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

> 



#### sporty-pub-prod-bi-bigdata-instance-1

> MySQL Community instance with Primary + Replica setup. Storage at ~90% capacity.

[**sporty-pub-prod-bi-bigdata-instance-1 (Primary)**](https://eu-central-1.console.aws.amazon.com/cloudwatch/home?region=eu-central-1#metricsV2?graph=~(view~'timeSeries~stacked~false~region~'eu-central-1~start~'-PT2160H~end~'P0D)&query=~'*7bAWS*2fRDS*2cDBInstanceIdentifier*7d*20sporty-pub-prod-bi-bigdata-instance-1)

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

[**sporty-pub-prod-bi-bigdata-instance-2 (Replica)**](https://eu-central-1.console.aws.amazon.com/cloudwatch/home?region=eu-central-1#metricsV2?graph=~(view~'timeSeries~stacked~false~region~'eu-central-1~start~'-PT2160H~end~'P0D)&query=~'*7bAWS*2fRDS*2cDBInstanceIdentifier*7d*20sporty-pub-prod-bi-bigdata-instance-2)

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

> Serverless v2 (40–100 ACUs)

[**bigdata-ticket-prod-instance-1 (Writer)**](https://eu-central-1.console.aws.amazon.com/cloudwatch/home?region=eu-central-1#metricsV2?graph=~(view~'timeSeries~stacked~false~region~'eu-central-1~start~'-PT2160H~end~'P0D~stat~'Average~period~60)&query=~'*7bAWS*2fRDS*2cDBInstanceIdentifier*7d*20*20sporty-global-prod-bet-bi-instance-1)

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

[**bigdata-ticket-prod-instance-2 (Reader)**](https://eu-central-1.console.aws.amazon.com/cloudwatch/home?region=eu-central-1#metricsV2?graph=~(view~'timeSeries~stacked~false~region~'eu-central-1~start~'-PT2160H~end~'P0D~stat~'Average~period~60)&query=~'*7bAWS*2fRDS*2cDBInstanceIdentifier*7d*20bigdata-ticket-prod-instance-2)

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

### 3.2 Sporty UAT

#### sporty-pub-uat-bi-main2

> Aurora cluster with Writer + Reader separation. db.t4g.medium (burstable, 4 GB RAM). Metrics show two distinct phases: active (4/19–4/22) and near-idle (4/23 onwards).

[**sporty-pub-uat-bi-main2-instance-1 (Writer)**](https://eu-central-1.console.aws.amazon.com/cloudwatch/home?region=eu-central-1#metricsV2?graph=~(view~'timeSeries~stacked~false~region~'eu-central-1~start~'-PT2160H~end~'P0D~stat~'Average~period~60)&query=~'*7bAWS*2fRDS*2cDBInstanceIdentifier*7d*20sporty-pub-uat-bi-main2-instance-1)

| Metric | Avg | Peak | Risk |
|---|---|---|---|
| CPU Utilization | ~25–30% (active) / ~10–13% (post 4/22) | ~64.9% | Medium |
| DB Connections | ~6–9 (active) / ~0 (post 4/22) | ~18 | Low |
| Freeable Memory | ~741 MB–1.1 GB (active) / ~1.35 GB (post 4/22) | min ~741 MB | **High** |
| Read IOPS | ~0 (between spikes) | ~3.51K | Low |
| Write IOPS | ~40–70 (active) / ~0 (post 4/22) | ~131.92 | Low |

[**sporty-pub-uat-bi-main2-instance-2 (Reader)**](https://eu-central-1.console.aws.amazon.com/cloudwatch/home?region=eu-central-1#metricsV2?graph=~(view~'timeSeries~stacked~false~region~'eu-central-1~start~'-PT2160H~end~'P0D~stat~'Average~period~60)&query=~'*7bAWS*2fRDS*2cDBInstanceIdentifier*7d*20sporty-pub-uat-bi-main2-instance-2)

| Metric          | Avg                             | Peak        | Risk     |
| --------------- | ------------------------------- | ----------- | -------- |
| CPU Utilization | ~20–30%                         | ~64%        | Medium   |
| DB Connections  | ~5–8                            | ~21         | Low      |
| Freeable Memory | ~880 MB–1.08 GB (↑ trending up) | min ~779 MB | **High** |
| Read IOPS       | ~0 (daily spike)                | ~4.11K      | Low      |
| Write IOPS      | ~20–50 (replication)            | ~435.22     | Low      |

**Notes**
- Memory utilization ~80% (min freeable ~779 MB on 4 GB total) — high for a UAT instance
- Daily ReadIOPS spikes (~4.11K) indicate a batch read pipeline routing reads to this Reader consistently
- Anomalous WriteIOPS spike on 4/28 (~435.22) — significantly above baseline replication writes, needs investigation
- Notable contrast: instance-1 (Writer) went near-idle after 4/22 while this Reader remains consistently active — UAT writes have stopped but something continues reading from this cluster

#### sporty-global-uat-bet-bi

> Aurora Serverless v2 (0.5–20 ACUs) with single instance.

[**sporty-global-uat-bet-bi-instance-1**](https://eu-central-1.console.aws.amazon.com/cloudwatch/home?region=eu-central-1#metricsV2?graph=~(view~'timeSeries~stacked~false~region~'eu-central-1~start~'-PT2160H~end~'P0D~stat~'Average~period~60)&query=~'*7bAWS*2fRDS*2cDBInstanceIdentifier*7d*20sporty-global-uat-bet-bi-instance-1)

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

[**encore-global-prod-bet-bi-instance-1 (Reader)**](https://eu-central-1.console.aws.amazon.com/cloudwatch/home?region=eu-central-1#metricsV2?graph=~(view~'timeSeries~stacked~false~region~'eu-central-1~start~'-PT2160H~end~'P0D~stat~'Average~period~60)&query=~'*7bAWS*2fRDS*2cDBInstanceIdentifier*7d*20encore-global-prod-bet-bi-instance-1)

| Metric | Avg | Peak | Risk |
|---|---|---|---|
| CPU Utilization | ~2.21% | ~5.8% | Low |
| DB Connections | ~38–40 (baseline) | ~90 | Medium |
| Freeable Memory | ~39 GB | min ~28.29 GB | Low |
| Read IOPS | ~0 | ~8.76 | — |
| Write IOPS | ~0 | ~0 | — |

[**encore-global-prod-bet-bi-instance-2 (Writer)**](https://eu-central-1.console.aws.amazon.com/cloudwatch/home?region=eu-central-1#metricsV2?graph=~(view~'timeSeries~stacked~false~region~'eu-central-1~start~'-PT2160H~end~'P0D~stat~'Average~period~60)&query=~'*7bAWS*2fRDS*2cDBInstanceIdentifier*7d*20encore-global-prod-bet-bi-instance-2)

| Metric | Avg | Peak | Risk |
|---|---|---|---|
| CPU Utilization | ~2.38% (daily spike ~18–21%) | ~21.6% | Low |
| DB Connections | ~0 | ~2 | — |
| Freeable Memory | ~39 GB | min ~27.88 GB | Low |
| Read IOPS | ~0 (daily spike) | ~816.56 | Low |
| Write IOPS | ~0 (daily spike) | ~705.16 | Low |

**Notes**
- Clear daily batch pipeline pattern — both CPU and IOPS spike once per day then return to baseline
- DB Connections near-zero on Writer (0–2, brief) — batch job connects, runs, disconnects cleanly
- Instance-1 (Reader) maintains ~38–40 persistent idle connections with near-zero ReadIOPS — likely a connection pool holding connections without active queries; investigate which service is connecting
- Serverless ACU scaling visible in memory: daily dips to ~28 GB as ACUs scale up during batch run then release

### 3.4 Encore UAT

#### encore-pub-uat-bi-main

> Aurora cluster (db.t4g.medium, 4 GB RAM, burstable), single instance.

[**encore-pub-uat-bi-main-instance-1 (Writer)**](https://eu-central-1.console.aws.amazon.com/cloudwatch/home?region=eu-central-1#metricsV2?graph=~(view~'timeSeries~stacked~false~region~'eu-central-1~start~'-PT2160H~end~'P0D~stat~'Average~period~60)&query=~'*7bAWS*2fRDS*2cDBInstanceIdentifier*7d*20encore-pub-uat-bi-main-instance-1)

| Metric | Avg | Peak | Risk |
|---|---|---|---|
| CPU Utilization | ~10–13% (continuous) | ~19.1% | Medium |
| DB Connections | ~0 | ~3 | — |
| Freeable Memory | ~885–925 MB | min ~867 MB | **High** |
| Read IOPS | ~0 | ~0.15 | — |
| Write IOPS | ~4–12 (continuous) | ~26.95 | Low |

**Notes**
- Sustained CPU at 10–13% with near-zero connections and zero ReadIOPS is anomalous — likely a background process; needs investigation
- Notable behaviour change on 4/21: WriteIOPS spiked to 26.95 then settled at a new sustained baseline of ~9–12 IOPS through 4/27 before dropping back — investigate what changed
- Memory utilization ~78% (freeable ~867–925 MB on 4 GB) — high for a UAT instance with minimal active connections
- db.t4g.medium is a burstable instance; sustained CPU at 10–13% will drain CPU credits over time, risking CPU throttling

#### encore-global-uat-bet-bi

> Aurora Serverless v2 (0.5–20 ACUs), single instance.

[**encore-global-uat-bet-bi-instance-1 (Writer)**](https://eu-central-1.console.aws.amazon.com/cloudwatch/home?region=eu-central-1#metricsV2?graph=~(view~'timeSeries~stacked~false~region~'eu-central-1~start~'-PT2160H~end~'P0D~stat~'Average~period~60)&query=~'*7bAWS*2fRDS*2cDBInstanceIdentifier*7d*20encore-global-uat-bet-bi-instance-1)

| Metric | Avg | Peak | Risk |
|---|---|---|---|
| CPU Utilization | ~2.64–2.84% | ~3.03% | Low |
| DB Connections | ~0 | ~1 | — |
| Freeable Memory | ~38.19–38.83 GB (cyclical) | min ~38.19 GB | Low |
| Read IOPS | N/A (not captured) | — | — |
| Write IOPS | ~3.37–3.61 (background) | ~3.98 | Low |

**Notes**
- Instance is essentially idle — near-zero connections throughout the observation period (only 2 brief spikes to 1 connection on 4/22–4/25, none after)
- WriteIOPS (~3.37–3.61) represents Aurora background storage maintenance, not actual user writes
- FreeableMemory shows a regular ~5–7 day sawtooth cycle — consistent with Serverless ACU scale-up/scale-down behaviour; no risk
- Recommend reviewing whether this UAT instance is still needed given it has had no active usage

---

### 中文備註彙整

**sporty-pub-prod-bi-main**
- instance-1 (Reader)
	- Reader 記憶體峰值達約 82%（最低可用 ~70.73 GB / 384 GB 總量）——週期性尖峰與 pipeline 執行時間相關
	- ReadIOPS 在 4/23 尖峰至 15.4K——需調查是哪個 pipeline 造成
	- Writer WriteIOPS 峰值 63.81K——寫入負載重，需監控是否出現持續性尖峰

**sporty-pub-prod-bi-bigdata-instance-1**
- instance-1 (Primary)
	- 記憶體使用率約 86%（32 GB 總量，可用記憶體持續約 4.4 GB）→ 若承受額外負載有 OOM 風險
	- 連線數極低（平均 1–2）——值得確認是哪些服務在連線
	- 儲存空間 11,518 / 12,784 GiB（約 90% 已滿）→ 緊急，需立即處理
- instance-2 (Replica)
	- Replica 連線數及 ReadIOPS 幾乎為零——看起來沒有實際承擔讀取流量
	- 可用記憶體呈明顯下降趨勢（兩週內從 4.41 GB → 4.19 GB）→ 疑似緩慢記憶體洩漏，若不處理有 OOM 風險
	- 值得評估此 Replica 是否仍有存在必要

**bigdata-ticket-prod**
- instance-1 (Writer)
	- CPU 平均 40–60%，峰值 82.8%——可能已接近 Serverless 100 ACU 上限，解釋了持續不穩定的原因
	- WriteIOPS 持續維持在 10–20K 是造成 HLL（History List Length）問題的主要根因
	- 4/29–30 前後的 ReadIOPS 尖峰（15.29K）可能與 Blue/Green 切換作業有關
	- 記憶體深度下降（最低可用 ~44.85 GB）與高寫入負載期間相關
	- 正在遷移至 provisioned r8g.8xlarge——預期可改善穩定性與成本
- instance-2 (Reader)
	- 讀取爆量時記憶體可用量降至 ~23.86 GB——比 Writer 更嚴重
	- 讀取模式極度突發（尖峰至 16.54K 後驟降至 0）——與 batch pipeline 讀取時間相關
	- CPU 在重度讀取期間峰值達 60.2%

**sporty-global-prod-bet-bi**
- instance-1 (Reader)
	- 連線數（平均 ~600–1,000）對一台 Serverless 2–40 ACU instance 來說異常偏高——值得調查是哪些服務在連線
	- 記憶體週期性下降至 ~33.43 GB——與連線及讀取尖峰時間相關
- instance-2 (Writer)
	- CPU / WriteIOPS 呈明顯的每日一次規律性尖峰——符合每日排程 batch pipeline 的特徵
	- Writer 連線數極低（平均 0–1）——僅有 batch 存取，無互動式查詢
	- 4/20 異常事件：CPU 達 63%、記憶體降至 ~33 GB、WriteIOPS 達 3.61K——需調查當天是否有非常規 pipeline 執行
	- instance-1 Reader（~600–1,000 連線）與 instance-2 Writer（~0）之間的強烈對比——需確認是什麼在持續佔用 Reader 的連線

**sporty-pub-uat-bi-main2**
- instance-1 (Writer)
	- active 期間（4/19–4/22）記憶體使用率約 82%（4 GB 總量，最低可用 ~741 MB）——對 UAT instance 來說偏高
	- 4/22 之後 Writer 幾乎完全閒置——UAT 的寫入活動已停止，需確認原因
- instance-2 (Reader)
	- 每日 ReadIOPS 尖峰（~4.11K）顯示有 batch 讀取 pipeline 持續將讀取流量導向此 Reader
	- 4/28 WriteIOPS 異常尖峰（~435.22）——遠高於正常的 replication 寫入量，需調查
	- Writer 在 4/22 後閒置，但 Reader 仍持續有流量——UAT 寫入已停止但仍有服務在讀取

**sporty-global-uat-bet-bi**
- instance-1 (Writer)
	- 連線數（~140–170）對一台幾乎沒有讀寫活動的 UAT Serverless instance 來說異常偏高——很可能是 connection pool 未正確釋放所造成的閒置連線
	- 雖然連線數高，ReadIOPS 幾乎為零——連線皆處於 sleeping/idle 狀態，沒有實際查詢在執行

**encore-pub-prod-bi-main-v5-cluster**
- encore-pub-prod-bi-main-v5 (Reader)
	- CPU 峰值達 71%，ReadIOPS 持續維持在 4–6K——讀取壓力相當大
- encore-pub-prod-bi-main-v5-instance-2 (Writer)
	- Writer CPU 持續維持在 30–50%，峰值 82.3%——對 db.r6g.2xlarge 來說令人擔憂；5/02 起 CPU 有上升趨勢
	- Writer WriteIOPS 全天持續維持在 2–4K——重度連續寫入負載，峰值 11.23K
	- Cluster 整體風險：兩台 instance 均高度負載；可能需要升級 instance 規格（建議考慮 db.r6g.4xlarge 或更大）

**encore-global-prod-bet-bi**
- instance-1 (Reader)
	- 維持 ~38–40 個持續的閒置連線，但 ReadIOPS 幾乎為零——很可能是 connection pool 在無實際查詢的情況下佔用連線；需調查是哪個服務在連線
- instance-2 (Writer)
	- 明顯的每日 batch pipeline 模式——CPU 和 IOPS 每天各有一次尖峰後回到基準值
	- Writer 連線數幾乎為零（0–2，短暫）——batch 任務連線、執行、乾淨斷線
	- 記憶體中可見 Serverless ACU scaling 的行為：每日 batch 執行時記憶體降至 ~28 GB，結束後回升

**encore-pub-uat-bi-main**
- instance-1 (Writer)
	- 在幾乎沒有連線和零 ReadIOPS 的情況下，CPU 持續維持在 10–13% 屬於異常——很可能有背景 process 在運行；需調查
	- 4/21 有明顯的行為轉變：WriteIOPS 尖峰至 26.95 後，4/22–4/27 維持在新的持續基準值 ~9–12 IOPS，之後才回落——需調查當時發生了什麼變化
	- 記憶體使用率約 78%（4 GB 總量，可用 ~867–925 MB）——對幾乎沒有主動連線的 UAT instance 來說偏高
	- db.t4g.medium 是可突發型 instance；長期維持 10–13% CPU 會持續消耗 CPU Credits，有被 throttle 的風險

**encore-global-uat-bet-bi**
- instance-1 (Writer)
	- Instance 基本上處於閒置狀態——觀察期間連線數幾乎為零（僅在 4/22–4/25 有兩次短暫的 1 個連線，之後全無）
	- WriteIOPS（~3.37–3.61）代表 Aurora 儲存層的背景維護寫入，並非實際使用者寫入
	- FreeableMemory 呈現規律的 ~5–7 天鋸齒波形——符合 Serverless ACU scale-up/scale-down 的正常行為；無風險
	- 建議評估此 UAT instance 是否仍有存在必要，因為觀察期間幾乎沒有實際使用

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

RDS 歷史故事

Step1. 一開始只有 `bi-report-o1.mysql.pub.s.sportybet` 一台，類似現在 Redshift 的warehouse 功能，會直接從服務的 RDS 透過 cron job 將資料同步到這裡
Step2. 新增了 `bigdata-o1.mysql.pub.s.sportybet` 作為 serverless 的使用情境，每 15 分鐘會將資料同步過來，主要是給 Trading team 使用的
Step3. 大部分的工作從 cron job 遷移到 Airflow 上主要 pipeline 在 [data_analysis](https://github.com/opennetltd/data_analysis) 的 repo 裡面
Step4. 新開了一個 `bigdata-ticket-o1.mysql.pub.s.sportybet` 分擔 `bi-report-o1.mysql.pub.s.sportybet` 的 workload 

其他環境也都類似情境，現在大部分的工作應該都遷移到 Airflow 上了