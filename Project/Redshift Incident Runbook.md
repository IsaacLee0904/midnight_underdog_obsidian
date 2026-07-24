tags: #OpenNet  #data-engineering #redshift #data-warehouse 

---

This runbook covers incidents where Redshift slows down, stalls, or shuts down or heavy workload. It is organized into two parts : Part 1 covers how to investigate when a problem is detected, and Part 2 covers how to work through the backlog once Redshift is back.

# **Overview**

# **Part 1 : Detecting & Diagnosing**

## Early Signs

通常當 Slack alert channel 出現大量的 Airflow job 失敗錯誤時，很有可能就是 Redshift 出現了問題。當 Redshift 變慢或卡住時，查詢 Redshift 的 Airflow job 會開始大量 timeout / fail，這些失敗會被推送到幾個 Slack 告警頻道——**短時間內湧入大量 DAG fail 告警**，就是最早、也最常見的訊號。

## Triage 

## Diagnosis

## Immediate Mitigation



# **Part 2 : Recovery & Backlog Catch-up**



