# Background

BI RDS and Redshift sometimes run into long-running queries. [A recent example](https://opennetltd.slack.com/archives/C065GNMGECU/p1779245375502009 "https://opennetltd.slack.com/archives/C065GNMGECU/p1779245375502009"), on May 20, 2026, a query ran for 10+ hours on Aurora MySQL, pushing HLL to 1.1M and slowing down SELECT performance across the board. Without hard limits in place, this kind of query can drag down DB performance and affect downstream services. This document goes through the existing query execution limits across all instances and flags where no hard limit is set.
![[Pasted image 20260617150147.png]]

# Query Hard Limits Inventory

## Redshift
![[Pasted image 20260617150200.png]]

For Redshift, we run both Provisioned and Serverless clusters. Heavy DA jobs run on the Provisioned cluster, most of lighter DA jobs use the `bi-report` serverless workgroup, DS jobs use `data-science`, and warehouse sync jobs — except frequent syncs (< 15 min interval) which run on Provisioned — use `data-quality`. For query limits, Provisioned clusters use WLM queue rules, Serverless workgroups enforce limits via the Maximum Query Execution Time configuration.

### Redshift Provisioned

Redshift uses [Workload Management (WLM)](https://eu-central-1.console.aws.amazon.com/redshiftv2/home?region=eu-central-1#/workload-management?parameter-group=sporty-pub-prod-bi-warehouse "https://eu-central-1.console.aws.amazon.com/redshiftv2/home?region=eu-central-1#/workload-management?parameter-group=sporty-pub-prod-bi-warehouse") to manage query resources — queries get routed to different queues based on DB user or query group. Each queue can set an `abort_long_running_queries` rule to automatically kill queries that run too long.

![[Pasted image 20260617150214.png]]

#### [Sporty Prod bi_warehouse](https://eu-central-1.console.aws.amazon.com/redshiftv2/home?region=eu-central-1#/workload-management?parameter-group=sporty-pub-prod-bi-warehouse "https://eu-central-1.console.aws.amazon.com/redshiftv2/home?region=eu-central-1#/workload-management?parameter-group=sporty-pub-prod-bi-warehouse")

|   |   |   |
|---|---|---|
|**Queue**|**User Role / Query Group**|**Hard Limit**|
|Monitor Queue|`app_redshift_exporter` (integrate Grafana metrics)|None|
|OpenMetadata Queue|`app_openmetadata` (OM Usage)|None|
|Human user queries|`ds_group`, `da_group`, `de_group`, `de_select`, `trading_group` (personal account)|300s|
|Metabase queries|`metabase` (Metabase query)|100s|
|BI DAG long queries|query group: `bi_dag_long_queries` (DA Airflow)|3600s|
|BI DAG queries|`bi_report` (DA Airflow)|1200s|
|DS queue|`app_ai` (DS Airflow)|None|
|Default queue|all others (DE Airflow)|None|

#### [Sporty Prod bi_report](https://eu-central-1.console.aws.amazon.com/redshiftv2/home?region=eu-central-1#/workload-management?parameter-group=sporty-pub-prod-data-analysis "https://eu-central-1.console.aws.amazon.com/redshiftv2/home?region=eu-central-1#/workload-management?parameter-group=sporty-pub-prod-data-analysis")

|   |   |   |
|---|---|---|
|**Queue**|**User Role / Query Group**|**Hard Limit**|
|Human user queries|`ds_group`, `da_group`, `de_group`, `de_select` (personal account)|300s|
|Metabase queries|`metabase` (Metabase query)|100s|
|BI DAG queries|`bi_report` (DA Airflow)|5400s|
|Default queue|all others (DE Airflow)|None|

#### [Sporty UAT bi_warehouse / bi_report](https://eu-central-1.console.aws.amazon.com/redshiftv2/home?region=eu-central-1#/workload-management?parameter-group=sporty-pub-uat-bi-warehouse "https://eu-central-1.console.aws.amazon.com/redshiftv2/home?region=eu-central-1#/workload-management?parameter-group=sporty-pub-uat-bi-warehouse")

|   |   |   |
|---|---|---|
|**Queue**|**User Role / Query Group**|**Hard Limit**|
|Monitor Queue|`app_redshift_exporter` (integrate Grafana metrics)|None|
|OpenMetadata Queue|`app_openmetadata` (OM Usage)|None|
|DS queue|`app_ai` (DS Airflow)|None|
|BI DAG long queries|query group: `bi_dag_long_queries` (DA Airflow)|None|
|Default queue|all others|None|

#### [Encore Prod bi_warehouse / bi_report](https://eu-central-1.console.aws.amazon.com/redshiftv2/home?region=eu-central-1#/workload-management?parameter-group=encore-pub-prod-bi-warehouse "https://eu-central-1.console.aws.amazon.com/redshiftv2/home?region=eu-central-1#/workload-management?parameter-group=encore-pub-prod-bi-warehouse")

|   |   |   |
|---|---|---|
|**Queue**|**User Role / Query Group**|**Hard Limit**|
|Monitor Queue|`app_redshift_exporter` (integrate Grafana metrics)|None|
|OpenMetadata Queue|`app_openmetadata` (OM Usage)|None|
|Metabase queries|`metabase` (Metabase query)|200s|
|Default queue|all others|None|

#### Encore UAT bi_warehouse / bi_report

|   |   |   |
|---|---|---|
|**Queue**|**User Role / Query Group**|**Hard Limit**|
|Monitor Queue|`app_redshift_exporter` (integrate Grafana metrics)|None|
|OpenMetadata Queue|`app_openmetadata` (OM Usage)|None|
|Metabase queries|`metabase` (Metabase query)|100s|
|Default queue|all others|None|

### Redshift Serverless

For Serverless workgroups, query execution limits are configured at the workgroup level via the `Maximum Query Execution Time` setting. Queries exceeding this limit are automatically terminated.

|   |   |   |
|---|---|---|
|**Serverless Workgroup**|**Usage**|**Max Query Execution Time**|
|sporty-pub-prod-bi-data-quality-workgroup|Sporty Prod Warehouse Airflow (non-frequent sync)|14400s|
|sporty-pub-prod-bi-report-workgroup|Sporty Prod DA Airflow (lighter jobs)|3600s|
|sporty-pub-prod-bi-data-science-workgroup|Sporty Prod DS Airflow|14400s|
|sporty-pub-uat-bi-data-quality-workgroup|Sporty UAT Warehouse Airflow (non-frequent sync)|14400s|
|sporty-pub-uat-bi-report-workgroup|Sporty UAT DA Airflow (lighter jobs)|600s|
|sporty-pub-uat-bi-data-science-workgroup|Sporty UAT DS Airflow|14400s|

### Airflow-level Mechanisms

#### Auto-kill

1. `dagrun_timeout`
    - When a DAG run times out, running tasks are forcibly stopped and marked as `skipped`, and the DAG run is marked as `failed`.
    - ⚠️ This only kills the Airflow task but the Redshift query is not terminated and may continue running as a zombie.
    - Reference : [Airflow monitoring DAGs | check_and_kill_accidental_queries](https://opennetltd.atlassian.net/wiki/spaces/DET/pages/3938025503/Airflow+monitoring+DAGs#check_and_kill_accidental_queries)
        
2. `check_and_kill_accidental_queries`
    - Detects queries running longer than 5 minutes where the query starts with `SELECT *` and contains `ORDER BY` + `LIMIT` ( typical TablePlus table preview behavior). Matching queries are automatically killed and a Slack alert is sent.
        

## RDS

### Database Configuration

Aurora MySQL enforces a hard limit on `SELECT` query execution time via the `max_execution_time` variable. Queries exceeding the limit are automatically terminated. Note that this setting does not apply to DML operations (`INSERT`, `UPDATE`, `DELETE`).

![[Pasted image 20260617150238.png]]
Reference : [Aurora MySQL Configuration Parameter](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/AuroraMySQL.Reference.ParameterGroups.html "https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/AuroraMySQL.Reference.ParameterGroups.html")

|**Instance**|**Env**|**max_execution_time**|
|---|---|---|
|sporty-pub-prod-bi-main|Sporty Prod|7200000ms (2hr)|
|sporty-pub-prod-bi-bigdata-instance-1|Sporty Prod|7200000ms (2hr)|
|bigdata-ticket-prod|Sporty Prod|7200000ms (2hr)|
|sporty-global-prod-bet-bi|Sporty Prod|7200000ms (2hr)|
|sporty-global-uat-bet-bi|Sporty UAT|7200000ms (2hr)|
|sporty-pub-uat-bi-main2|Sporty UAT|7200000ms (2hr)|
|encore-pub-prod-bi-main-v5-cluster|Encore Prod|7200000ms (2hr)|
|encore-global-prod-bet-bi|Encore Prod|7200000ms (2hr)|
|encore-pub-uat-bi-main|Encore UAT|7200000ms (2hr)|
|encore-global-uat-bet-bi|Encore UAT|7200000ms (2hr)|

Reference :
- [DBA-11859: Check and adjust the BI related RDSs global 'max_execution_time' limit.In Progress](https://opennetltd.atlassian.net/browse/DBA-11859)
- [SRE-10736: modify the RDS parameter ‘max_execution_time’ to ‘7200000’Done](https://opennetltd.atlassian.net/browse/SRE-10736)

### Airflow-level Mechanisms

#### Auto-kill

1. `dagrun_timeout`
    - When a DAG run times out, running tasks are forcibly stopped and marked as `skipped`, and the DAG run is marked as `failed`.
    - ⚠️ This only kills the Airflow task but the Redshift query is not terminated and may continue running as a zombie.
    - Reference : [Airflow monitoring DAGs | check_and_kill_accidental_queries](https://opennetltd.atlassian.net/wiki/spaces/DET/pages/3938025503/Airflow+monitoring+DAGs#check_and_kill_accidental_queries)
        
2. `metabase_long_running_queries_monitor`
    - Every 10 minutes, automatically kills Metabase queries that are stuck or running too long on all BI RDS instances (except Encore UAT).
    - Reference : [Metabase Long Running Queries Monitor](https://opennetltd.atlassian.net/wiki/spaces/DET/pages/3241115649)
        

# Gap Summary

### 🔴 RDS only has SELECT execution time limit

#### **Current State**

Now we only applied `max_execution_time` configuration for RDS and that only works for `SELECT` queries.

#### **Potential solution**

1. **Monitoring DAG**
    * Create an Airflow DAG that periodically scans `information_schema.processlist` and kills queries (including DML) exceeding a defined execution time threshold, similar to the existing `check_and_kill_accidental_queries` pattern.
    * Pros : No further infrastructure needed and consistent with existing pattern (Redshift), highly customizable such as whitelist, thresholds, alerting
    * Cons : Minimum interval ~1 min (limit of Airflow DAG), cannot catch very short-lived queries
    * Another similar implementation with [MySQL Event Scheduler](https://blogs.reliablepenguin.com/2025/09/28/automatically-killing-long-running-queries-in-aurora-mysql-serverless-v2 "https://blogs.reliablepenguin.com/2025/09/28/automatically-killing-long-running-queries-in-aurora-mysql-serverless-v2")
        
2. **pt-kill (Percona Toolkit)**
    * A command-line tool that continuously monitors and kills queries based on execution time, user, or query pattern. Requires a dedicated host to run.
    * Pros : Fine-grained control over which queries to kill (by user, database, or query pattern)
    * Cons : Requires further machine to run
    * reference : [Percona Toolkit Documentation](https://docs.percona.com/percona-toolkit/pt-kill.html)
        
### 🟡 Some of **Redshift queues without hard limits**
- DS queue and Default queue on Sporty Prod have no execution time limit
- Sporty UAT and Encore clusters have almost no limits across all queues