## Overview
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

# 2. Instance Inventory & Performance Assessment

This section documents all BI RDS instances across Sporty and Encore environments (Prod and UAT). For each instance, configuration details are included - engine version, endpoint (listed as Route53 alias where available, otherwise raw AWS endpoint), instance type, storage, and Multi-AZ setup — along with performance metrics collected via CloudWatch and cost data where applicable. Known issues and observations will be consolidated in the Issues & Risks Summary section. Instance list was referenced from the DBA Confluence page BI Related RDS List and cross-checked against Airflow connections across all environments (Prod/UAT × Sporty/Encore × Warehouse/DA).

### 2.1 Sporty PROD

#### [sporty-pub-prod-bi-main](https://eu-central-1.console.aws.amazon.com/rds/home?region=eu-central-1#database:id=sporty-pub-prod-bi-main;is-cluster=true)

Cluster has Read / Write separation — instance-1 handles read queries, instance-2 (Writer) handles all write traffic. Originally served as the primary BI data warehouse; now functions as a data mart, receiving transformed data from upstream Airflow pipelines and serving read queries from the DA team and Metabase.

1. **Endpoint**：
   - `bi-report-o1.mysql.pub.s.sportybet` (writer)
   - `bigdata-o1.mysql.pub.s.sportybet` (writer)
   - `replica-bigdata-o1` (reader / DA Airflow)
2. **Engine**：Aurora MySQL 8.0.mysql_aurora.3.10.1
3. **Instance Type**：db.r6g.12xlarge
4. **Storage**：Aurora Standard ( auto-scaling )
5. **Multi-AZ**：Yes ( 2 Zones )
6. **Cost (Q1 2026)**：Jan $29,196 / Feb $24,266 / Mar $26,477 （avg ~$26,646/mo）
7. **CloudWatch Monitoring**
	* sporty-pub-prod-bi-main-instance-1
		* Performance Insights：Enabled（7 days retention）
		* Enhanced Monitoring：Enabled（60s）
	* sporty-pub-prod-bi-main-instance-2
		* Performance Insights：Enabled（7 days retention）
		* Enhanced Monitoring：Enabled（60s）

[**sporty-pub-prod-bi-main-instance-1 (Reader)**](https://eu-central-1.console.aws.amazon.com/cloudwatch/home?region=eu-central-1#metricsV2?graph=~(view~'timeSeries~stacked~false~region~'eu-central-1~start~'-PT2160H~end~'P0D)&query=~'*7bAWS*2fRDS*2cDBInstanceIdentifier*7d*20sporty-pub-prod-bi-main-instance-1)

| Metric | Avg | Peak | Risk |
|---|---|---|---|
| CPU Utilization | ~5–10% | ~39.5% | Low |
| DB Connections | ~30–50 | ~105 | Low |
| Freeable Memory | ~120–141 GB | min ~70.73 GB | Medium |
| Read IOPS | ~1–2K | ~15.4K | Low |
| Write IOPS | ~0 | ~0 | — |

[**sporty-pub-prod-bi-main-instance-2 (Writer)**](https://eu-central-1.console.aws.amazon.com/cloudwatch/home?region=eu-central-1#metricsV2?graph=~(view~'timeSeries~stacked~false~region~'eu-central-1~start~'-PT2160H~end~'P0D)&query=~'*7bAWS*2fRDS*2cDBInstanceIdentifier*7d*20sporty-pub-prod-bi-main-instance-2)

| Metric          | Avg         | Peak        | Risk   |
| --------------- | ----------- | ----------- | ------ |
| CPU Utilization | ~20–30%     | ~41.7%      | Low    |
| DB Connections  | ~8–16       | ~49         | Low    |
| Freeable Memory | ~128–132 GB | min ~114 GB | Low    |
| Read IOPS       | ~1–2K       | ~13.88K     | Low    |
| Write IOPS      | ~15–25K     | ~63.81K     | Medium |

**Notes**
- RDS Extended Support enabled → incurring additional cost
- Reader memory peaks at ~82% (min freeable ~70.73 GB / 384 GB total) — periodic spikes correlate with pipeline runs
- Reader ReadIOPS spike to 15.4K on 4/23 — investigate which pipeline caused this
- Writer WriteIOPS peak at 63.81K — heavy write load, worth monitoring for sustained spikes

---
#### [sporty-pub-prod-bi-bigdata-instance-1](https://eu-central-1.console.aws.amazon.com/rds/home?region=eu-central-1#database:id=sporty-pub-prod-bi-bigdata-instance-1;is-cluster=false)

MySQL Community instance with Primary + Replica setup. Originally created to offload workload from bi-report-o1, but is largely unused now — only a few specific pipelines remain on this instance.

1. **Endpoint**：
	* `bi-bigdata-o1.mysql.pub.s.sportybet`
	* `replica-bi-bigdata-o1.mysql.pub.s.sportybet` (Primary, Airflow Conn)
2. **Engine**：MySQL Community 8.0.40
3. **Instance Type**：db.r6g.xlarge
4. **Storage**：gp3 / 11,518 GiB / 12,000 IOPS
5. **Multi-AZ**：Yes
6. **Cost (Q1 2026)**：Jan $5,952 / Feb $5,916 / Mar $6,090 （avg ~$5,986/mo）

[**sporty-pub-prod-bi-bigdata-instance-1 (Primary)**](https://eu-central-1.console.aws.amazon.com/cloudwatch/home?region=eu-central-1#metricsV2?graph=~(view~'timeSeries~stacked~false~region~'eu-central-1~start~'-PT2160H~end~'P0D)&query=~'*7bAWS*2fRDS*2cDBInstanceIdentifier*7d*20sporty-pub-prod-bi-bigdata-instance-1)

| Metric          | Avg     | Peak         | Risk     |
| --------------- | ------- | ------------ | -------- |
| CPU Utilization | ~20–25% | ~32.8%       | Low      |
| DB Connections  | ~1–2    | ~21          | Low      |
| Freeable Memory | ~4.5 GB | min ~4.39 GB | **High** |
| Read IOPS       | ~800–1K | ~4.73K       | Low      |
| Write IOPS      | ~3–4K   | ~6.25K       | Low      |

[**sporty-pub-prod-bi-bigdata-instance-2 (Replica)**](https://eu-central-1.console.aws.amazon.com/cloudwatch/home?region=eu-central-1#metricsV2?graph=~(view~'timeSeries~stacked~false~region~'eu-central-1~start~'-PT2160H~end~'P0D)&query=~'*7bAWS*2fRDS*2cDBInstanceIdentifier*7d*20sporty-pub-prod-bi-bigdata-instance-2)

| Metric | Avg | Peak | Risk |
|---|---|---|---|
| CPU Utilization | ~2–4% | ~18.9% | Low |
| DB Connections | ~0 | ~2 | — |
| Freeable Memory | ~4.2–4.4 GB (↓ trending) | min ~4.19 GB | **High** |
| Read IOPS | ~0 | ~525 | Low |
| Write IOPS | ~300–600 | ~3.78K | Low |

**Notes**
- Storage at 11,518 / 12,784 GiB (~90% full) → urgent, needs attention
- Performance Insights disabled → monitoring gap
- Enhanced Monitoring disabled → monitoring gap
- Memory utilization ~86% (32 GB total, freeable consistently ~4.4 GB) → risk of OOM under additional load
- Very low connection count (avg 1–2) — instance is largely unused; confirm which pipelines are still connected
- Replica has near-zero connections and read IOPS — appears to be unused for actual read traffic
- Freeable Memory on a clear downward trend (4.41 GB → 4.19 GB over 2 weeks) → potential slow memory leak, risk of OOM if not addressed
- Worth evaluating whether this instance and its Replica are still needed

---
#### [bigdata-ticket-prod](https://eu-central-1.console.aws.amazon.com/rds/home?region=eu-central-1#database:id=bigdata-ticket-prod;is-cluster=true)

Aurora cluster that syncs data every 15 minutes, primarily serving the Trading team.

1. **Endpoint**：
    - `bigdata-ticket-o1.mysql.pub.s.sportybet` (writer, Airflow Conn)
    - `bigdata-ticket-o1.mysql.ro.pub.s.sportybet` (reader)
2. **Engine**：Aurora MySQL 8.0.mysql_aurora.3.10.1
3. **Instance Type**：db.r8g.8xlarge (provisioned)
4. **Storage**：Aurora Standard (auto-scaling)
5. **Multi-AZ**：No
6. **Cost (Q1 2026)**：Jan $18,129.94 / Feb $16,689.96 / Mar $19,215.20 （avg ~$18,012/mo）

[**bigdata-ticket-prod-instance-1(Writer)**](https://eu-central-1.console.aws.amazon.com/cloudwatch/home?region=eu-central-1#metricsV2?graph=~(metrics~(~(~'AWS*2fRDS~'DatabaseConnections~'DBInstanceIdentifier~'sporty-global-prod-bet-bi-instance-1~(id~'m1~visible~false))

| Metric          | Avg             | Peak         | Risk     |
| --------------- | --------------- | ------------ | -------- |
| CPU Utilization | ~30–50%         | ~71.7%       | **High** |
| DB Connections  | ~20–30          | ~100         | Medium   |
| Freeable Memory | ~19–21 GB       | min ~8.85 GB | **High** |
| Read IOPS       | ~0–500 (bursty) | ~11.52K      | Medium   |
| Write IOPS      | ~10–20K         | ~79.89K      | **High** |

[**bigdata-ticket-prod-instance-2 (Reader)**](https://eu-central-1.console.aws.amazon.com/cloudwatch/home?region=eu-central-1#metricsV2?graph=~(view~'timeSeries~stacked~false~region~'eu-central-1~start~'-PT2160H~end~'P0D~stat~'Average~period~60)&query=~'*7bAWS*2fRDS*2cDBInstanceIdentifier*7d*20bigdata-ticket-prod-instance-2)

| Metric          | Avg    | Peak          | Risk   |
| --------------- | ------ | ------------- | ------ |
| CPU Utilization | ~5–10% | ~37.5%        | Low    |
| DB Connections  | ~9–10  | ~19           | Low    |
| Freeable Memory | ~28 GB | min ~16.48 GB | Medium |
| Read IOPS       | N/A    | N/A           | -      |
| Write IOPS      | ~0     | ~0            | -      |

**Notes**
- Migration from Serverless to provisioned (r8g.8xlarge) in progress — DBA-7596
- Recurring HLL (History List Length) issues — root cause under investigation
- CPU avg 40–60% with peaks at 82.8% — likely approaching Serverless 100 ACU ceiling, explains recurring instability
- Write IOPS consistently 10–20K is the primary driver of HLL issues; 15-minute sync cadence means writes never stop
- ReadIOPS spike to 15.29K around 4/29–30 likely related to Blue/Green switchover activities
- Memory deep dips (~44.85 GB freeable) correlate with high write load periods
- Reader memory dips to ~23.86 GB during read bursts — more severe than Writer
- Migration to provisioned r8g.8xlarge in progress — expected to improve stability and cost

---
#### [sporty-global-prod-bet-bi](https://eu-central-1.console.aws.amazon.com/rds/home?region=eu-central-1#database:id=sporty-global-prod-bet-bi;is-cluster=true)

1. **Endpoint**：`bet-bi-o1.mysql.global.s.sportybet` (writer, Airflow Conn)
2. **Engine**：8.0.mysql_aurora.3.10.1
3. **Instance Type**：Serverless v2 (2–40 ACUs)
4. **Storage**：Aurora Standard (auto-scaling)
5. **Multi-AZ**：2 Zones
6. **Cost (Q1 2026)**：Jan $806.28 / Feb $796.46 / Mar $824.78（avg ~$809/mo）

[sporty-global-prod-bet-bi-instance-1](https://eu-central-1.console.aws.amazon.com/cloudwatch/home?region=eu-central-1#metricsV2?graph=~(view~'timeSeries~stacked~false~region~'eu-central-1~start~'-PT2160H~end~'P0D~stat~'Average~period~60)&query=~'*7bAWS*2fRDS*2cDBInstanceIdentifier*7d*20sporty-global-prod-bet-bi-instance-1)

| Metric          | Avg         | Peak       | Risk                                                    |
| --------------- | ----------- | ---------- | ------------------------------------------------------- |
| CPU Utilization | ~5–15%      | ~39%       | Low                                                     |
| DB Connections  | ~700–1,400  | ~1.4K      | Dropped to ~40 after 05/19                              |
| Freeable Memory | ~67–77 GB   | min ~38 GB | Low                                                     |
| Read IOPS       | ~100–400 /s | ~1.54K     | Low                                                     |
| Write IOPS      | ~0          | ~2K        | Sudden spike on 05/20, near-zero for preceding 3 months |

[sporty-global-prod-bet-bi-instance-2](https://eu-central-1.console.aws.amazon.com/cloudwatch/home?region=eu-central-1#metricsV2?graph=~(view~'timeSeries~stacked~false~region~'eu-central-1~start~'-PT2160H~end~'P0D~stat~'Average~period~60)&query=~'*7bAWS*2fRDS*2cDBInstanceIdentifier*7d*20sporty-global-prod-bet-bi-instance-2)

| Metric          | Avg                                                        | Peak       | Risk                                        |
| --------------- | ---------------------------------------------------------- | ---------- | ------------------------------------------- |
| CPU Utilization | ~0 (burst spike daily) → ~5–15% after 05/19                | ~56.2%     | Low                                         |
| DB Connections  | ~0 before 05/19 → ~455–910 after 05/19                     | ~910       | Jumped from near-zero to active after 05/19 |
| Freeable Memory | ~67–77 GB                                                  | min ~38 GB | Low                                         |
| Read IOPS       | ~0 (burst spike daily) → continuous ~100–200/s after 05/19 | ~1.58K     | Low                                         |
| Write IOPS      | Daily burst spikes up to ~3.38K before 05/19 → ~0 after    | ~3.38K     | —                                           |

⚠️ Connections appear to have been redirected from instance-1 to instance-2 around 05/19. Needs further investigation.

#### [sporty-global-stg-bet-bi](https://eu-central-1.console.aws.amazon.com/rds/home?region=eu-central-1#database:id=sporty-global-stg-bet-bi;is-cluster=true)

Reverse ETL target. BI Airflow pipelines write processed data to this cluster for backend application consumption. Instance management is not owned BI.

---
### 2.2 Sporty UAT

#### [sporty-global-uat-bet-bi](https://eu-central-1.console.aws.amazon.com/rds/home?region=eu-central-1#database:id=sporty-global-uat-bet-bi;is-cluster=true)

Aurora Serverless v2 (0.5–20 ACUs), single instance (no Reader/Writer split).

1. **Endpoint**：
	* `bet-bi-t1.mysql.global.s.sportybet` (writer, Airflow Conn)
	* `sporty-global-uat-bet-bi.cluster-ro-cvdgobmslfew.eu-central-1.rds.amazonaws.com`
2. **Engine**：Aurora MySQL 8.0.mysql_aurora.3.04.3
3. **Instance Type**：Serverless v2 (0.5–20 ACUs)
4. **Storage**：Aurora Standard (auto-scaling)
5. **Multi-AZ**：No
6. **Cost (Q1 2026)**：Jan $52.24 / Feb $43.89 / Mar $56.99（avg ~$51/mo）

[**sporty-global-uat-bet-bi-instance-1**](https://eu-central-1.console.aws.amazon.com/cloudwatch/home?region=eu-central-1#metricsV2?graph=~(view~'timeSeries~stacked~false~region~'eu-central-1~start~'-PT2160H~end~'P0D~stat~'Average~period~60)&query=~'*7bAWS*2fRDS*2cDBInstanceIdentifier*7d*20sporty-global-uat-bet-bi-instance-1)

| Metric          | Avg       | Peak          | Risk     |
| --------------- | --------- | ------------- | -------- |
| CPU Utilization | ~2.9%     | ~4.63%        | Low      |
| DB Connections  | ~140–170  | ~183          | **High** |
| Freeable Memory | ~36–38 GB | min ~30.18 GB | Low      |
| Read IOPS       | ~0        | ~0.03         | —        |
| Write IOPS      | ~3.79     | ~8.4          | Low      |

**Notes**
- Engine version 3.04.3 — significantly behind other instances (3.10.1) → upgrade needed
- RDS Extended Support enabled → incurring additional cost
- Deletion protection disabled
- Enhanced Monitoring disabled → monitoring gap
- DB Connections (~140–170) is unexpectedly high for a UAT Serverless instance with near-zero read/write activity — likely idle connections held by a connection pool that is not releasing properly
- ReadIOPS essentially zero despite high connection count — connections are sleeping/idle, no actual queries being executed
---
#### [sporty-pub-uat-bi-main2](https://eu-central-1.console.aws.amazon.com/rds/home?region=eu-central-1#database:id=sporty-pub-uat-bi-main2;is-cluster=true)

Aurora cluster with Writer + Reader separation. db.t4g.medium (burstable, 4 GB RAM). Metrics show two distinct phases: active (4/19–4/22) and near-idle (4/23 onwards).

1. **Endpoint**：
	* `bi-main2-t1.mysql.pub.s.sportybet` (writer, Airflow Conn)
	* `sporty-pub-uat-bi-main2.cluster-ro-cvdgobmslfew.eu-central-1.rds.amazonaws.com`
2. **Engine**：Aurora MySQL 8.0.mysql_aurora.3.10.1
3. **Instance Type**：db.t4g.medium
4. **Storage**：Aurora Standard (auto-scaling)
5. **Multi-AZ**：Yes (2 Zones)
6. **Cost (Q1 2026)**：Jan $74.35 / Feb $65.95 / Mar $100.28（avg ~$80/mo）

[**sporty-pub-uat-bi-main2-instance-1 (Writer)**](https://eu-central-1.console.aws.amazon.com/cloudwatch/home?region=eu-central-1#metricsV2?graph=~(view~'timeSeries~stacked~false~region~'eu-central-1~start~'-PT2160H~end~'P0D~stat~'Average~period~60)&query=~'*7bAWS*2fRDS*2cDBInstanceIdentifier*7d*20sporty-pub-uat-bi-main2-instance-1)

| Metric | Avg | Peak | Risk |
|---|---|---|---|
| CPU Utilization | ~25–30% (active) / ~10–13% (post 4/22) | ~64.9% | Medium |
| DB Connections | ~6–9 (active) / ~0 (post 4/22) | ~18 | Low |
| Freeable Memory | ~741 MB–1.1 GB (active) / ~1.35 GB (post 4/22) | min ~741 MB | **High** |
| Read IOPS | ~0 (between spikes) | ~3.51K | Low |
| Write IOPS | ~40–70 (active) / ~0 (post 4/22) | ~131.92 | Low |

[**sporty-pub-uat-bi-main2-instance-2 (Reader)**](https://eu-central-1.console.aws.amazon.com/cloudwatch/home?region=eu-central-1#metricsV2?graph=~(view~'timeSeries~stacked~false~region~'eu-central-1~start~'-PT2160H~end~'P0D~stat~'Average~period~60)&query=~'*7bAWS*2fRDS*2cDBInstanceIdentifier*7d*20sporty-pub-uat-bi-main2-instance-2)

| Metric | Avg | Peak | Risk |
|---|---|---|---|
| CPU Utilization | ~20–30% | ~64% | Medium |
| DB Connections | ~5–8 | ~21 | Low |
| Freeable Memory | ~880 MB–1.08 GB (↑ trending up) | min ~779 MB | **High** |
| Read IOPS | ~0 (daily spike) | ~4.11K | Low |
| Write IOPS | ~20–50 (replication) | ~435.22 | Low |

**Notes**
- RDS Extended Support enabled → incurring additional cost
- Enhanced Monitoring disabled → monitoring gap
- Writer CPU at 30.27% on db.t4g.medium (burstable instance) → may hit CPU credit limits under load
- Memory utilization ~80% (min freeable ~779 MB on 4 GB total) — high for a UAT instance
- Daily ReadIOPS spikes (~4.11K) indicate a batch read pipeline routing reads to this Reader consistently
- Anomalous WriteIOPS spike on 4/28 (~435.22) — significantly above baseline replication writes, needs investigation
- Notable contrast: instance-1 (Writer) went near-idle after 4/22 while this Reader remains consistently active — UAT writes have stopped but something continues reading from this cluster

---

### 2.3 Encore PROD

#### [encore-pub-prod-bi-main-v5-cluster](https://eu-central-1.console.aws.amazon.com/rds/home?region=eu-central-1#database:id=encore-pub-prod-bi-main-v5-cluster;is-cluster=true)

Aurora cluster (db.r6g.2xlarge, 64 GB RAM) with Reader + Writer separation. Serves as the primary BI warehouse for the Encore pub environment.

1. **Endpoint**：
   * `bi-main-o1.mysql.pub.s.encore` (writer, Airflow Conn)
   * `bi-main-o1.mysql.ro.pub.s.encore` (reader)
1. *Engine**：Aurora MySQL 8.0.mysql_aurora.3.10.1
2. **Instance Type**：db.r6g.2xlarge
3. **Storage**：Aurora Standard (auto-scaling)
4. **Multi-AZ**：Yes (2 Zones)
5. **Cost (Q1 2026)**：Jan $4,098.08 / Feb $3,534.69 / Mar $5,072.35（avg ~$4,235/mo）

[**encore-pub-prod-bi-main-v5 (Reader)**](https://eu-central-1.console.aws.amazon.com/cloudwatch/home?region=eu-central-1#metricsV2?graph=~(view~'timeSeries~stacked~false~region~'eu-central-1~start~'-PT2160H~end~'P0D~stat~'Average~period~60)&query=~'*7bAWS*2fRDS*2cDBInstanceIdentifier*7d*20*20encore-pub-prod-bi-main-v5)

| Metric          | Avg     | Peak          | Risk     |
| --------------- | ------- | ------------- | -------- |
| CPU Utilization | ~25–40% | ~71%          | **High** |
| DB Connections  | ~8–17   | ~51           | Low      |
| Freeable Memory | ~29 GB  | min ~25.59 GB | Low      |
| Read IOPS       | ~4–6K   | ~10.54K       | Medium   |
| Write IOPS      | ~0      | ~0            | —        |

[**encore-pub-prod-bi-main-v5-instance-2 (Writer)**](https://eu-central-1.console.aws.amazon.com/cloudwatch/home?region=eu-central-1#metricsV2?graph=~(view~'timeSeries~stacked~false~region~'eu-central-1~start~'-PT2160H~end~'P0D~stat~'Average~period~60)&query=~'*7bAWS*2fRDS*2cDBInstanceIdentifier*7d*20encore-pub-prod-bi-main-v5-instance-2)

| Metric | Avg | Peak | Risk |
|---|---|---|---|
| CPU Utilization | ~30–50% | ~82.3% | **High** |
| DB Connections | ~4–8 | ~25 | Low |
| Freeable Memory | ~25.87–26.67 GB | min ~23.39 GB | Low |
| Read IOPS | ~200–500 | ~2.86K | Low |
| Write IOPS | ~2–4K | ~11.23K | **High** |

**Notes**
- RDS Extended Support enabled → incurring additional cost
- Enhanced Monitoring disabled → monitoring gap
- Writer CPU consistently at 30–50% with peak at 82.3% — concerning for a db.r6g.2xlarge; CPU trend appears to be increasing from 5/02 onwards
- Writer WriteIOPS sustained at 2–4K around the clock — heavy continuous write load, peak 11.23K
- Reader CPU peaks at 71% with sustained ReadIOPS 4–6K — both instances under significant pressure
- Cluster-wide concern: both instances are heavily utilized; may need instance type upgrade (consider db.r6g.4xlarge or larger)

---

#### [encore-global-prod-bet-bi](https://eu-central-1.console.aws.amazon.com/rds/home?region=eu-central-1#database:id=encore-global-prod-bet-bi;is-cluster=true)

Aurora Serverless v2 (0.5–20 ACUs) cluster with Reader + Writer separation. Serves global Encore bet data.

1. **Endpoint**：
   * `bet-bi-o1.mysql.global.s.encore` (writer, Airflow Conn)
   * `encore-global-prod-bet-bi.cluster-ro-cvdgobmslfew.eu-central-1.rds.amazonaws.com` (reader)
2. **Engine**：Aurora MySQL 8.0.mysql_aurora.3.10.1
3. **Instance Type**：Serverless v2 (0.5–20 ACUs)
4. **Storage**：Aurora Standard (auto-scaling)
5. **Multi-AZ**：Yes (2 Zones)
6. **Cost (Q1 2026)**：Jan $106.47 / Feb $98.02 / Mar $109.02（avg ~$104/mo）

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
- RDS Extended Support enabled → incurring additional cost
- Enhanced Monitoring disabled → monitoring gap
- Clear daily batch pipeline pattern — both CPU and IOPS spike once per day then return to baseline
- DB Connections near-zero on Writer (0–2, brief) — batch job connects, runs, disconnects cleanly
- Instance-1 (Reader) maintains ~38–40 persistent idle connections with near-zero ReadIOPS — likely a connection pool holding connections without active queries; investigate which service is connecting
- Serverless ACU scaling visible in memory: daily dips to ~28 GB as ACUs scale up during batch run then release

---

### 2.4 Encore UAT

#### [encore-pub-uat-bi-main](https://eu-central-1.console.aws.amazon.com/rds/home?region=eu-central-1#database:id=encore-pub-uat-bi-main;is-cluster=true)

Aurora cluster (db.t4g.medium, 4 GB RAM, burstable), single Writer instance.

1. **Endpoint**：
   * `bi-main-t1.mysql.pub.s.encore` (writer)
   * `encore-pub-uat-bi-main.cluster-ro-cvdgobmslfew.eu-central-1.rds.amazonaws.com` (reader)
2. **Engine**：Aurora MySQL 8.0.mysql_aurora.3.04.3
3. **Instance Type**：db.t4g.medium
4. **Storage**：Aurora Standard (auto-scaling)
5. **Multi-AZ**：No
6. **Cost (Q1 2026)**：Jan $36.06 / Feb $32.59 / Mar $36.27（avg ~$35/mo）

[**encore-pub-uat-bi-main-instance-1**](https://eu-central-1.console.aws.amazon.com/cloudwatch/home?region=eu-central-1#metricsV2?graph=~(view~'timeSeries~stacked~false~region~'eu-central-1~start~'-PT2160H~end~'P0D~stat~'Average~period~60)&query=~'*7bAWS*2fRDS*2cDBInstanceIdentifier*7d*20encore-pub-uat-bi-main-instance-1)

| Metric | Avg | Peak | Risk |
|---|---|---|---|
| CPU Utilization | ~10–13% (continuous) | ~19.1% | Medium |
| DB Connections | ~0 | ~3 | — |
| Freeable Memory | ~885–925 MB | min ~867 MB | **High** |
| Read IOPS | ~0 | ~0.15 | — |
| Write IOPS | ~4–12 (continuous) | ~26.95 | Low |

**Notes**
- Engine version 3.04.3 — significantly behind other instances (3.10.1) → upgrade needed
- RDS Extended Support enabled → incurring additional cost
- No Multi-AZ, single Writer only
- Enhanced Monitoring disabled → monitoring gap
- Sustained CPU at 10–13% with near-zero connections and zero ReadIOPS is anomalous — likely a background process; needs investigation
- Notable behaviour change on 4/21: WriteIOPS spiked to 26.95 then settled at a new sustained baseline of ~9–12 IOPS through 4/27 before dropping back — investigate what changed
- Memory utilization ~78% (freeable ~867–925 MB on 4 GB) — high for a UAT instance with minimal active connections
- db.t4g.medium is a burstable instance; sustained CPU at 10–13% will drain CPU credits over time, risking CPU throttling

---

#### [encore-global-uat-bet-bi](https://eu-central-1.console.aws.amazon.com/rds/home?region=eu-central-1#database:id=encore-global-uat-bet-bi;is-cluster=true)

Aurora Serverless v2 (0.5–20 ACUs), single instance.

1. **Endpoint**
   * `bet-bi-t1.mysql.global.s.encore` (reader)
   * `encore-global-uat-bet-bi.cluster-ro-cvdgobmslfew.eu-central-1.rds.amazonaws.com` (writer)
1. **Engine**：Aurora MySQL 8.0.mysql_aurora.3.04.3
2. **Instance Type**：Serverless v2 (0.5–20 ACUs)
3. **Storage**：Aurora Standard (auto-scaling)
4. **Multi-AZ**：No
5. **Cost (Q1 2026)**：Jan $46.74 / Feb $42.25 / Mar $58.71（avg ~$49/mo）

[**encore-global-uat-bet-bi-instance-1**](https://eu-central-1.console.aws.amazon.com/cloudwatch/home?region=eu-central-1#metricsV2?graph=~(view~'timeSeries~stacked~false~region~'eu-central-1~start~'-PT2160H~end~'P0D~stat~'Average~period~60)&query=~'*7bAWS*2fRDS*2cDBInstanceIdentifier*7d*20encore-global-uat-bet-bi-instance-1)

| Metric          | Avg                        | Peak          | Risk |
| --------------- | -------------------------- | ------------- | ---- |
| CPU Utilization | ~2.64–2.84%                | ~3.03%        | Low  |
| DB Connections  | ~0                         | ~1            | —    |
| Freeable Memory | ~38.19–38.83 GB (cyclical) | min ~38.19 GB | Low  |
| Read IOPS       | N/A (not captured)         | —             | —    |
| Write IOPS      | ~3.37–3.61 (background)    | ~3.98         | Low  |

**Notes**
- Engine version 3.04.3 — significantly behind other instances (3.10.1) → upgrade needed
- RDS Extended Support enabled → incurring additional cost
- No Multi-AZ, single Writer only
- Enhanced Monitoring disabled → monitoring gap
- Instance is essentially idle — near-zero connections throughout the observation period (only 2 brief spikes to 1 connection on 4/22–4/25, none after)
- WriteIOPS (~3.37–3.61) represents Aurora background storage maintenance, not actual user writes
- FreeableMemory shows a regular ~5–7 day sawtooth cycle — consistent with Serverless ACU scale-up/scale-down behaviour; no risk
- Recommend reviewing whether this UAT instance is still needed given it has had no active usage

---

# 3. DAG Review

### 3.1 Airflow Connection

[Google Sheet - Airflow Connection List](https://docs.google.com/spreadsheets/d/13ZaFzEW6pqTMLzs2TqH2AdGH-1C8XTWIssnm2s2Tl7Y/edit?gid=0#gid=0)

### 3.2 DAG Overview

There are three typical data flow pattern to read or write those BI related RDS :

- Direct RDS read / write : Majority of DAGs read with bi_report_rds_conn_ro and write back via bi_report_rds_conn.
- Redshift → S3 → RDS : Used by some reporting pipelines such as GGR, user segmentation etc.
- Application DB → RDS : Some pipeline read from per-country application DB directorly and write to RDS without passing by S3.
    
For the full list of all Airflow DAGs please refer to [DAG List tab](https://docs.google.com/spreadsheets/d/13ZaFzEW6pqTMLzs2TqH2AdGH-1C8XTWIssnm2s2Tl7Y/edit?usp=sharing) in the GoogleSheet.

### 3.3 Know Issues

1. **Credential Storage Inconsistency** : bigdata_ticket_prod (Sporty DA Prod), bi_bigdata_ro, and bigdata_ticket_prod_ro (Encore DA Prod) are stored in AWS Secrets Manager rather than the Airflow Connection UI. To reduce confusion, should migrate all connections to a unified storage approach.
2. **Partial Migration to** app_airflow **Account** : A planned migration to all BI RDS connections to use the app_airflow account is in progress. The bi-bigdata connections on Sporty Airflow have been completed.  
    ref : [https://opennetltd.atlassian.net/browse/BDE-1121?atlOrigin=eyJpIjoiNzVkZWIyNjczYWY3NGZmM2I1YTczNWYwOTIyZjc2MjgiLCJwIjoiaiJ9](https://opennetltd.atlassian.net/browse/BDE-1121?atlOrigin=eyJpIjoiNzVkZWIyNjczYWY3NGZmM2I1YTczNWYwOTIyZjc2MjgiLCJwIjoiaiJ9)
3. **RW Connections Used for Read-Only Tasks** : Some DAGs doing read-only operations through write-capable connections. This does not follow the intended connection type and would adds unnecessary load on write endpoints.

# 4. Monitoring Review
RDS monitoring is centralized through a [DBA-managed Grafana stack](https://opennetltd.atlassian.net/wiki/spaces/DBA/pages/4169170997/Aurora+MySQL+Monitor+System+Architecture), with metrics sourced from CloudWatch Streams and direct MySQL exporters. Alerts are delivered to Slack.

![[Pasted image 20260617152902.png]]

### 4.1 Monitoring Coverage

TBD

### 4.2 Alerting Coverage

RDS alerting rules are defined in the [dba-alert-rules](https://github.com/opennetltd/dba-alert-rules/tree/main/aurora-mysql) repo, as documented in [MySQL Alerting rules system](https://opennetltd.atlassian.net/wiki/spaces/DBA/pages/3286728768/MySQL+Alerting+rules+system).

![[Screenshot 2026-06-17 at 6.06.49 PM.png|639]]
All alerts are delivered to Slack (`dba-alert`) via Grafana.