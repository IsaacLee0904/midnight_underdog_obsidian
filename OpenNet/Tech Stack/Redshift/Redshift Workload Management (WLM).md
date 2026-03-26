WLM 是 Redshift 用來**管理查詢資源分配**的機制，核心概念是把不同類型的查詢放進不同的 queue，避免一個重型查詢拖垮整個 cluster

## Basic Info


## OpenNet WLM Structure

### Overview
![[Redshift Workload Management|800]]
Link : [Workload Management Page](https://eu-central-1.console.aws.amazon.com/redshiftv2/home?region=eu-central-1#/workload-management?parameter-group=sporty-pub-prod-bi-warehouse)

### Detail of different Queue

大致上是利用 <span style="color:rgb(255, 0, 0)">db user account</span> 來決定 query 會決定進去哪一個 queue

#### Monitor Queue

1. **用途** : 用來 integrate Grafana metrics 用的 queue
2. **Query priority** : Highest
3. **User role** : app_redshift_exporter

#### Openmetatadata Queue

1. **用途** : OM 用的 queue，主要提供 DQ check 與 data linage 的功能
2. **Query priority** : Highest
3. **User role** : app_redshift_exporter

#### Human user queries

#### Metabase queries

#### BI DAG long queries

#### BI DAG queries

#### DS queue

#### Default queue
