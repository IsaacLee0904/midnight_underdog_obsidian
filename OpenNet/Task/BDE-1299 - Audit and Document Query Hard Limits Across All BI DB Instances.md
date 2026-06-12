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


---


根據我交叉比對了所有相關文件，以下是完整的盤點結果：

---

## 所有 BI RDS 實例清單

根據 [BI related RDS list](https://opennetltd.atlassian.net/wiki/spaces/DBA/pages/4432626286)Preview 和你自己寫的 [BI RDS Inventory and Current State (2026)](https://opennetltd.atlassian.net/wiki/spaces/DET/pages/4496195776)Preview，共 11 個 BI data instance：

**Sporty Prod (4)**

|   |   |   |   |
|---|---|---|---|
|#|Cluster|Engine|Instance Type|
|1|`sporty-pub-prod-bi-main`|Aurora MySQL 3.10.1|db.r6g.12xlarge|
|2|`sporty-pub-prod-bi-bigdata`|MySQL Community 8.0.40|db.r6g.xlarge|
|3|`bigdata-ticket-prod`|Aurora MySQL 3.10.1|db.r8g.8xlarge|
|4|`sporty-global-prod-bet-bi`|Aurora MySQL 3.10.1|Serverless v2|

**Sporty UAT (2)**  
| 5 | `sporty-global-uat-bet-bi` | Aurora MySQL 3.10.1 | Serverless v2 |  
| 6 | `sporty-pub-uat-bi-main2` | Aurora MySQL 3.10.1 | db.t4g.medium |

**Encore Prod (2)**  
| 7 | `encore-pub-prod-bi-main-v5-cluster` | Aurora MySQL 3.10.1 | db.r6g.2xlarge |  
| 8 | `encore-global-prod-bet-bi` | Aurora MySQL 3.10.1 | Serverless v2 |

**Encore UAT (2)**  
| 9 | `encore-pub-uat-bi-main` | Aurora MySQL 3.04.3 | db.t4g.medium |  
| 10 | `encore-global-uat-bet-bi` | Aurora MySQL 3.04.3 | Serverless v2 |

另外還有 `metabase-rds`（Metabase metadata backend），不算 data instance。

---

## Query Hard Limit 盤點結果

**結論：所有 BI RDS 實例都沒有任何 query execution time hard limit。**

|   |   |   |   |
|---|---|---|---|
|機制|當前設定|作用範圍|是否會 Kill Query|
|`max_execution_time`|**0（無限制）** — 全部實例|僅 SELECT|❌ 未啟用|
|`wait_timeout`|28800（8小時）|閒置連線|❌ 只斷開 idle session|
|`interactive_timeout`|28800（8小時）|互動式 idle|❌ 只斷開 idle session|
|`long_query_time`|5 秒|所有 query|❌ 只記 slow log，不 kill|
|`aurora_oom_response`|`print,tune`（預設）|記憶體壓力|⚠️ 只有 OOM 時才觸發|
|Metabase Monitor DAG|有自定閾值|僅 Metabase user|✅ 但只限 Metabase 來源|

來源：[Default parameter for new created RDS](https://opennetltd.atlassian.net/wiki/spaces/DBA/pages/2702540842)Preview、[RDS Parameter Group Configuration in Terraform](https://opennetltd.atlassian.net/wiki/spaces/DBA/pages/4493377563)Preview

---

## 為什麼 DBA-11781 事件沒被擋下來

[![](https://opennetltd.atlassian.net/rest/api/2/universal_avatar/view/type/issuetype/avatar/10318)DBA-11781: p0-dba-alert / AuroraMySQLRollbackSegmentHistoryListLength (sporty-pub-prod-bi-main-instance-2)Done](https://opennetltd.atlassian.net/browse/DBA-11781)Preview 的 root cause 是 DA 透過 **TablePlus** 直接連 `sporty-pub-prod-bi-main-instance-2` 跑了一個 1 年期的 heavy query（`st_sporty_games_daily_user_pl_eur` 全表 GROUP BY），跑了 10+ 小時造成 HLL 飆高。這個場景：

- `max_execution_time = 0` → 不會自動 kill
    
- Metabase Monitor → 不涵蓋 TablePlus 連線
    
- `aurora_oom_response` → 沒觸發 OOM 所以不會 kill
    
- `wait_timeout` → query 是 active 狀態不算 idle
    

**完全沒有任何機制可以攔截。**

---

## 覆蓋缺口

|   |   |   |
|---|---|---|
|查詢來源|是否有限制|缺口|
|Metabase（dashboard/探索）|✅ Airflow DAG 監控|無|
|Airflow DAG（ETL pipeline）|❌|無 timeout 設定|
|DA 個人工具（TablePlus/DBeaver）|❌|**這次事件的缺口**|
|Application account（app_airflow）|❌|無 timeout 設定|

---

要我直接把這個盤點結果寫入你正在編輯的 [Aurora MySQL Monitor System Architecture](https://opennetltd.atlassian.net/wiki/spaces/DBA/pages/4169170997)Preview 頁面嗎？或者你想先調整內容再加？

![jira](https://opennetltd.atlassian.net/gateway/api/assist/rovo/v1/resources/icon/jira)

3 Sources