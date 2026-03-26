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
3. **Concurrency on main** : Auto
4. **User role** : app_redshift_exporter

#### Openmetatadata Queue

1. **用途** : OM 用的 queue，主要提供 DQ check 與 data linage 的功能
2. **Query priority** : Low
3. **Concurrency on main** : Auto
4. **User role** : app_openmetadata

#### Human user queries

1. **用途** : 一般使用者 personal account 用的 queue
2. **Query priority** : Low
3. **Concurrency on main** : Auto
4. **User role** : ds_group, da_group, de_group, de_select

#### Metabase queries

1. **用途** : Metabase dashboard 使用的 queue
2. **Query priority** : Normal
3. **Concurrency on main** : Auto
4. **User role** : bi_dag_long_queries

#### BI DAG long queries

1. **用途** : DA Airflow DAG 使用的 queue
2. **Query priority** : Normal
3. **Concurrency on main** : Auto
4. **User role** : metabase

#### BI DAG queries

#### DS queue

#### Default queue
