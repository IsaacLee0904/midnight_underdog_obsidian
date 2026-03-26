WLM 是 Redshift 用來**管理查詢資源分配**的機制，核心概念是把不同類型的查詢放進不同的 queue，避免一個重型查詢拖垮整個 cluster

## Basic Info

1. Query priority : 決定了查詢優先執行的順序，Redshift 會決定是否給予更多資源來值行
2. Concurrency on main : 決定是否會丟到 concurrency scaling 去執行 (auto scaling 的概念，但因為原本的 Redshift 機器很大台，所以 auto scaling 現在也很大台)
3. user role : 決定什麼 user 會被丟進什麼 queue

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
5. **Query monitoring rules** : abort_long_running_queries (Query execution time (seconds) > 300)

#### Metabase queries

1. **用途** : Metabase dashboard 使用的 queue
2. **Query priority** : Normal
3. **Concurrency on main** : Auto
4. **User role** : bi_dag_long_queries
5. **Query monitoring rules** : abort_long_running_queries (Query execution time (seconds) > 100)

#### BI DAG long queries

1. **用途** : DA Airflow DAG 使用的 queue
2. **Query priority** : Normal
3. **Concurrency on main** : Auto
4. <mark style="background: #FFB86CA6;">Query groups</mark> : bi_dag_long_queries
5. **Query monitoring rules** : abort_long_running_queries (Query execution time (seconds) > 3600)

#### BI DAG queries

1. **用途** : DA Airflow DAG 使用的 queue
2. **Query priority** : Normal
3. **Concurrency on main** : Auto
4. **User role** : bi_report
5. **Query monitoring rules** : abort_long_running_queries (Query execution time (seconds) > 1200)

#### DS queue

1. **用途** : DS Airflow DAG 使用的 queue
2. **Query priority** : Normal
3. **Concurrency on main** : Auto
4. **User role** : app_ai
5. **Query monitoring rules** : abort_long_running_queries (Query execution time (seconds) > 3600)

#### Default queue

1. **用途** : DE Airflow DAG 使用的 queue
2. **Query priority** : Normal
3. **Concurrency on main** : Auto
4. **User role** : 不是上面的情況全部都會用這個
5. **Query monitoring rules** : abort_long_running_queries (Query execution time (seconds) > 5400)
