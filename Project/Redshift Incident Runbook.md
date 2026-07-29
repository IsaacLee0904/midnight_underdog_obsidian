tags: #OpenNet  #data-engineering #redshift #data-warehouse 

---
# **Overview**

This runbook covers incidents where Redshift slows down, stalls, or shuts down or heavy workload. It is organized into two parts : Part 1 covers how to investigate when a problem is detected, and Part 2 covers how to work through the backlog once Redshift is back.

## Platform Architecture

### Redshift 

#### Redshift Cluster Structure

目前 Redshift 採用的是 multi-cluster structure，下面以 Prod Sporty 為例，所有資料實際上存放在 `bi-warehouse` ，其他 consumer cluster 透過 data share 取得資料

![[Screenshot 2026-07-28 at 5.23.57 PM.png]]
![[Screenshot 2026-07-29 at 2.30.34 PM.png]]

#### Redshift Workload Manage

`bi_warehouse` WLM
![[Screenshot 2026-07-29 at 2.44.21 PM.png]]
ref : [[Redshift Workload Management (WLM)]]

`bi_analysis` WLM

目前是使用 auto WLM，搭配 session / user level 的 statement_timeout 

```sql
SET statement_timeout TO 600000; -- 600s
```

>[! warning]
>之前有做實驗嘗試用 manucal WLM，但 AWS 已經確認了 Redshift 內部會把一個查詢改成多個子查詢，變成 timeout 實際上是套用在每一個子查詢上而不是總時長，因此改回 auto

### Airflow

#### Airflow Routing Rules

目前透過 `readshift_multicluster_settings` Airflow variable 做 routing 來決定與方便控制不同的 DAG 要在哪一個 cluster 上面跑
![[Screenshot 2026-07-29 at 3.09.56 PM.png]]

Step1. DA job 預設使用 `_prod_ro` 或 `_prod_rw` 來作為 conn
Step2. 執行時會去讀取 `readshift_multicluster_settings` 比對 cluster 的 tags 與 DAG tags
Step3. 比對完之後會轉換到相對應的 `connection_mappings` 中的 cluster conn


For entire information please reference [DE internal briefing about Redshift, Airflow, Monitoring](https://opennetltd.atlassian.net/wiki/spaces/DET/pages/4578279470/DE+internal+briefing+about+Redshift+Airflow+Monitoring)
# **Part 1 : Detecting & Diagnosing**

## Early Signs

通常當 Slack alert channel 出現大量的 Airflow job 失敗錯誤時，很有可能就是 Redshift 出現了問題。當 Redshift 變慢或卡住時，查詢 Redshift 的 Airflow job 會開始大量 timeout / fail，這些失敗會被推送到幾個 Slack 告警頻道——**短時間內湧入大量 DAG fail 告警**，就是最早、也最常見的訊號。

## Triage 

## Diagnosis

## Immediate Mitigation



# **Part 2 : Recovery & Backlog Catch-up**


Routing Rules with Airflow Variable

**normal settings**

```json
[
  {
    "cluster_name": "bi-warehouse",
    "aws_redshift_iam_role": "arn:aws:iam::942878658013:role/sporty-pub-prod-redshift-warehouse",
    "tags": ["cluster:bi-warehouse", "management", "v12", "v13", ["1_day", "redshift"]],
    "enabled": true,
    "connection_mappings": [
      {
        "from_connection": "warehouse_bi_pub_prod_rw",
        "to_connection": "warehouse_bi_pub_prod_rw"
      },
      {
        "from_connection": "warehouse_pub_prod_ro",
        "to_connection": "warehouse_pub_prod_ro"
      },
      {
        "from_connection": "*",
        "to_connection": "warehouse_bi_pub_prod_rw"
      }
    ]
  },
  {
    "cluster_name": "bi-report",
    "aws_redshift_iam_role": "arn:aws:iam::942878658013:role/sporty-pub-prod-redshift-warehouse",
    "tags": ["cluster:bi-report", "adhoc", "backfill", "hqe"],
    "enabled": true,
    "connection_mappings": [
      {
        "from_connection": "warehouse_bi_pub_prod_rw",
        "to_connection": "warehouse_bi_pub_prod_bi_report_serverless_rw"
      },
      {
        "from_connection": "warehouse_pub_prod_ro",
        "to_connection": "warehouse_pub_prod_bi_report_serverless_ro"
      },
      {
        "from_connection": "*",
        "to_connection": "warehouse_bi_pub_prod_bi_report_serverless_rw"
      }
    ]
  },
  {
    "cluster_name": "data-analysis",                           
    "enabled": true,
    "tags": ["cluster:data-analysis", "high_importance", ["1_hour", "redshift"]],
    "aws_redshift_iam_role": "arn:aws:iam::942878658013:role/sporty-pub-prod-redshift-warehouse",
    "connection_mappings": [
      {
        "from_connection": "warehouse_pub_prod_ro",
        "to_connection": "warehouse_bi_data_analysis_ro"
      },
      {
        "from_connection": "*",
        "to_connection": "warehouse_bi_data_analysis_rw"
      }
    ]
  }
]
```


**serverless CPU spike setting**
```json
[
  {
    "cluster_name": "bi-warehouse",
    "aws_redshift_iam_role": "arn:aws:iam::942878658013:role/sporty-pub-prod-redshift-warehouse",
    "tags": ["cluster:bi-warehouse", "management", "v12", "v13", "booking_code", "high_importance", ["1_day", "redshift"]],
    "enabled": true,
    "connection_mappings": [
      {
        "from_connection": "warehouse_bi_pub_prod_rw",
        "to_connection": "warehouse_bi_pub_prod_rw"
      },
      {
        "from_connection": "warehouse_pub_prod_ro",
        "to_connection": "warehouse_pub_prod_ro"
      },
      {
        "from_connection": "*",
        "to_connection": "warehouse_bi_pub_prod_rw"
      }
    ]
  },
  {
    "cluster_name": "bi-report",
    "aws_redshift_iam_role": "arn:aws:iam::942878658013:role/sporty-pub-prod-redshift-warehouse",
    "tags": ["cluster:bi-report", "adhoc", "backfill", "hqe", ["1_week", "redshift"]],
    "enabled": true,
    "connection_mappings": [
      {
        "from_connection": "warehouse_bi_pub_prod_rw",
        "to_connection": "warehouse_bi_pub_prod_bi_report_serverless_rw"
      },
      {
        "from_connection": "warehouse_pub_prod_ro",
        "to_connection": "warehouse_pub_prod_bi_report_serverless_ro"
      },
      {
        "from_connection": "*",
        "to_connection": "warehouse_bi_pub_prod_bi_report_serverless_rw"
      }
    ]
  },
  {
    "cluster_name": "data-analysis", 
    "enabled": true,
    "tags": ["cluster:data-analysis", "high_importance", ["1_hour", "redshift"]],
    "aws_redshift_iam_role": "arn:aws:iam::942878658013:role/sporty-pub-prod-redshift-warehouse",
    "connection_mappings": [
      {
        "from_connection": "warehouse_pub_prod_ro",
        "to_connection": "warehouse_bi_data_analysis_ro"
      },
      {
        "from_connection": "*",
        "to_connection": "warehouse_bi_data_analysis_rw"
      }
    ]
  }
]
```