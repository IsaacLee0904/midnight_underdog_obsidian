tags: #OpenNet  #data-engineering #redshift #data-warehouse 

---
# **Overview**

This runbook covers incidents where Redshift slows down, stalls, or shuts down or heavy workload. It is organized into two parts : Part 1 covers how to investigate when a problem is detected, and Part 2 covers how to work through the backlog once Redshift is back.

## Platform Architecture



For entire information please reference [DE internal briefing about Redshift, Airflow, Monitoring](https://opennetltd.atlassian.net/wiki/spaces/DET/pages/4578279470/DE+internal+briefing+about+Redshift+Airflow+Monitoring)


# **Part 1 : Detecting & Diagnosing**

## Early Signs

通常當 Slack alert channel 出現大量的 Airflow job 失敗錯誤時，很有可能就是 Redshift 出現了問題。當 Redshift 變慢或卡住時，查詢 Redshift 的 Airflow job 會開始大量 timeout / fail，這些失敗會被推送到幾個 Slack 告警頻道——**短時間內湧入大量 DAG fail 告警**，就是最早、也最常見的訊號。

## Triage 

## Diagnosis

## Immediate Mitigation



# **Part 2 : Recovery & Backlog Catch-up**


Routing Rules with Airflow Variable

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