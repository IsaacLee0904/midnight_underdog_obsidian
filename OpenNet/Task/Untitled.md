根據搜尋結果，你們組織內已經有幾個相關機制，分層級整理如下：

## 1. Aurora 內建：`aurora_oom_response` 參數

當記憶體不足時，Aurora 可以自動 kill 佔用最多記憶體的 query。你們的 parameter group 文件中有詳細的設定建議，例如：

- OLTP 場景建議：`print, decline, kill_connect, tune_buffer_pool`
    
- Analytics 場景建議：`print, kill_query, tune_buffer_pool`
    

但這是 **記憶體壓力觸發**，不是單純基於 query 執行時間。[AWS Aurora RDS MySQL Parameter](https://opennetltd.atlassian.net/wiki/spaces/DBA/pages/3648782379/AWS+Aurora+RDS+MySQL+Parameter)Preview

## 2. MySQL 參數：`max_execution_time`

在 AHI Lock Contention 的 case study 中有提到可以設定：

`SET GLOBAL max_execution_time = 60000; -- 60s per query`

這會自動終止超過指定時間的 **SELECT** query（注意：只對 read-only SELECT 有效，不影響 DML）。[Case Study: Aurora MySQL AHI Lock Contention — sportybet-ng-prod-instant-win (2025-03-17)](https://opennetltd.atlassian.net/wiki/spaces/DBA/pages/4425777210)Preview

## 3. 自建 Airflow DAG：Metabase Long Running Queries Monitor

這是你們 DET 團隊針對 Metabase 建的自動化機制，透過 Airflow DAG：

1. 查詢 `INFORMATION_SCHEMA.PROCESSLIST`
    
2. 篩選出 Metabase 來源的 query
    
3. 超過閾值就自動 `KILL`
    
4. 發 Slack 通知到 `#bi_job`
    

[Metabase Long Running Queries Monitor](https://opennetltd.atlassian.net/wiki/spaces/DET/pages/3241115649/Metabase+Long+Running+Queries+Monitor)Preview

## 4. 其他相關參數

在 [RDS Parameter Group Configuration in Terraform](https://opennetltd.atlassian.net/wiki/spaces/DBA/pages/4493377563)Preview 中也有記錄：

- `long_query_time`：slow query log 的閾值（預設 10 秒），但這只是 **記錄** 不會 kill
    
- `interactive_timeout` / `wait_timeout`：閒置連線超時，不直接處理 long-running query
    

---

總結來說，目前 **沒有一個通用的、針對所有 user 的 auto-kill long-running query 機制**。`max_execution_time` 最接近你要的功能，但只對 SELECT 有效。如果要更全面的覆蓋，可能需要類似 Metabase monitor 那樣自建 DAG，或者在 parameter group 層級設定 `max_execution_time`。

需要我幫你把這些機制整理到你正在編輯的 Aurora MySQL Monitor System Architecture 頁面嗎？

![confluence](https://opennetltd.atlassian.net/gateway/api/assist/rovo/v1/resources/icon/confluence)

2 Sources