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
Step4. 如果沒有 match 到任何 tags 就會繼續在 `bi-warehouse` 跑
![[Screenshot 2026-07-29 at 3.36.50 PM.png]]

<mark style="background:#fff88f">Sporty Prod Setting</mark>
![[Screenshot 2026-07-30 at 1.44.09 PM.png]]
* `bi-warehouse`
	* 角色：預設會讓 Airflow job 跑在 `bi-warehouse` 
	* 規則：如果 tag 有 match "cluster:bi-warehouse", "management", "v12" 會跑在 `bi-warehouse` 上
* `data-analysis` 
	* 角色：作為主要負擔重要的 pipeline 與 hourly loading DAG 的任務
	* 規則：tag 有 "cluster:data-analysis" 或 "high_importance" 任意一個又或是同時有 "1_hour" + "redshift" 就會在 `data-analysis` 上跑
* `bi-report` (serverless) 
	* 角色：讓 Ad-hoc 分析、backfill 跑在這上面避免影響到 `data-anlaysis` 的 workload
	* 規則：如果 tag 裡面有 `"cluster:bi-report", "adhoc", "backfill", "hqe"`任意一個或是同時有 "1_day" + "redshift" 就會跑在這個 serverless 上

Airflow Variable Setting
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

#### Airflow Configuration Setting

##### Pool Setting
可以透過 [admin/pool](https://airflow-da-pub-prod-bi.on.sportybet2.com/pool/list/) 觀看目前有哪些 pool 以及 pool 的情況也可以透過 [Grafana dashboard](https://grafana-pub-prod-misc.k8s.on.sportybet2.com/d/ddvknf88xc3y8d/airflow-alerts?orgId=1&from=now-1h&to=now&timezone=utc) 中的 Default pool running slots 與 Default pool scheduled slots 來監控使用狀況

預設使用 `default_pool` (通用、slot 數量大) 來跑大多數的 job，當遇到重要或高並行的 job 則會透過在 task 中設定 `pool = <pool)name>`派到 slot 數較小的專用 pool (<font color="#ff0000">pool 越小代表刻意設定了可以並行的上限，用來保護 cluster</font>)

##### Worker Setting
目前有三個設定
* `PARALLELISM 128` ，代表跨所有 pool 與 worker 的全部最大同時執行 task 數是 128 
*  `WORKER_CONCURRENCY` 64 單一 Celery worker 可領取的最大 task 數
*  `MAX_ACTIVE_TASKS_PER_DAG 4` 單一 DAG run 內預設的最大並行 task 數，然後是可以透過在 DAG 中定義蓋掉的
  
```
Total System Capacity = WORKER_CONCURRENCY * Works count 
* 並且受到 PARALLELISM 的限制
```

##### Scheduler Setting
用來控制 scheduler 每個週期掃描多少 task instance 來決定接下來能跑什麼，目前設置是`MAX_TIS_PER_QUERY = 300`

⚠️ 如果設置的太低，低優先級的 task 無法被領取就會導致 task 卡在 scheduled 狀態，最終被 skip / fail ；但如果設置太高又會導致 db 負載過高

>[!NOTE] 設定 Worker or Scheduler ：data_service_deployement repo prod branch / [airflow-da/values-da.yaml](https://github.com/opennetltd/data_service_deployment/blob/prod/releases/airflow-da/values-da.yaml)

For entire information please reference [DE internal briefing about Redshift, Airflow, Monitoring](https://opennetltd.atlassian.net/wiki/spaces/DET/pages/4578279470/DE+internal+briefing+about+Redshift+Airflow+Monitoring)
# **Part 1 : Detecting & Diagnosing**

## Early Signs

通常當 Slack alert channel 出現大量的 Airflow job 失敗錯誤時，很有可能就是 Redshift 或 Airflow 出現了問題。當 Redshift 變慢或卡住時，查詢 Redshift 的 Airflow job 會開始大量 timeout / fail，這些失敗會被推送到幾個 Slack 告警頻道——**短時間內湧入大量 DAG fail 告警**，就是最早、也最常見的訊號

* bi_warehouse_alert
* bi_job
* bi_job_encore
* bi_high_priority_job
* bi_high_priority_job_encore

## Triage 

Early Signs 只告訴我們「一堆 job 掛了」，但還沒告訴我們「是哪一層的問題」。在衝進去 kill query 之前，先花一分鐘判斷方向，避免明明是 Airflow / 連線 / 程式碼的問題卻一直盯著 Redshift。

**先做幾個快速判斷：**

1. **範圍**：是**跨多個 DAG** 同時 fail，還是**集中在單一 / 剛部署**的 DAG？
	* 跨多個、不同 owner 一起掛 → 偏 Redshift 或共用資源層
	* 集中在單一 or 剛上線的 DAG → 偏該 DAG 的 code / config
2. **Airflow scheduled slots**：看 [Airflow Alerts dashboard](https://grafana-pub-prod-misc.k8s.on.sportybet2.com/d/ddvknf88xc3y8d/airflow-alerts?orgId=1&from=now-1h&to=now&timezone=utc)，scheduled slots 有沒有飆到數百？有 → task 塞住排不出去，偏 Redshift 變慢 / 卡住。
3. **錯誤訊息類型**：是 timeout、connection refused、auth error 還是 code exception？connection / auth 類 → 偏連線層（Redshift service outage、憑證被 GitHub workflow 改動）。

**四個可能的層級**（可能同時發生、互相觸發，非互斥）：

| Layer | 典型徵兆 |
|---|---|
| 🔴 **Redshift** | 長查詢卡住、CPU / Leader Node CPU 飆升、大量短查詢高並行耗盡 cluster |
| 🟡 **Airflow** | `MAX_TIS_PER_QUERY` 太低漏掉低優先級 task、新 DAG 高並行把其他 DAG 餓死 |
| 🔵 **Connection / Auth** | 目標 DB outage / 維護、憑證或權限被改、worker 與 cluster 間網路問題 |
| 🟣 **Code** | 共用 general function 有 bug、routing 被改壞 connection mapping、DAG 改動有副作用 |

> [!NOTE] 判斷結果
> 若判斷是 **🔴 Redshift 負載 / 效能** → 進 [[#Diagnosis]] 找出具體元凶。其他層級則跳出本 runbook 的主線，往對應方向處理。

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


Leader Node CPU spike setting**
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



### 第 11 頁 — 目前的監控：Airflow 與 Redshift

關鍵 dashboard、指標系統，以及排查系統劣化時要看的具體指標。

**Airflow 監控**

- **Default Pool Slots（Airflow Alerts Dashboard）** — [連結](https://grafana-pub-prod-misc.k8s.on.sportybet2.com/d/ddvknf88xc3y8d/airflow-alerts)
    - 追蹤指標：Running slots（正在執行的 task 數）、Scheduled slots（排隊等待 worker slot 的 task 數）。
    - 觀察重點：Redshift 劣化時，running 數維持在高檔、scheduled 數飆到數百 → task 堆積並失敗。
- **CPU / Memory 與 DAG Run Delay（Airflow EKS Dashboard）** — [連結](https://grafana-pub-prod-misc.k8s.on.sportybet2.com/d/adk6avn6n2tc0aee/airflow-eks)
    - 追蹤指標：各 DAG 的 CPU / Memory 使用率、DAG run delay（排定執行時間與實際開始的落差）。
    - 觀察重點：DAG 層級突然的 CPU/Mem 飆升代表 cluster 有問題或程式碼有問題。DAG run delay 是 pool 塞滿前的早期警訊。

**Redshift 監控**

- **AWS Console — Cluster Metrics**：CPU 使用率與 Disk 使用率 %、讀/寫 throughput、資料庫連線數。
- **AWS Console — Query Monitoring**：Queued vs. Running 查詢數、耗時辨識（檢查長時間查詢）。
- **Grafana — Redshift Cluster Dashboard** — [連結](https://grafana-infra.k8s.on.sportybet2.com/d/redshift-cluster/redshift-cluster)：整合式 dashboard，一頁呈現所有 cluster 層級基礎設施指標；事故時可直接與 Airflow 面板交叉比對。

---

### 第 12 頁 — 常見告警與根因

**Redshift 效能劣化的常見原因**——為什麼 cluster 會突然變慢、task 會 timeout？以下是擾亂運作的核心 pattern。

**🔄 持續 Backfill**

- 症狀：Redshift 出現長查詢、Airflow running slots 維持高檔、其他 job 變慢。
- 根因：新 job 在回補歷史資料，產生超出常態容量、未預期的大量查詢。
- 緩解：限制並行（`max_active_runs` 或專用 pool）／把 backfill 排在離峰時段／用 `"backfill"` tag 路由到 `bi-report`。

**⚡ 高並行 Job**

- 症狀：Redshift 連線數 / WLM queue 深度突然跳升、Airflow scheduled slots 飆升。
- 根因：新 job 同時送出大量平行查詢，壓垮 cluster。
- 緩解：用 `max_active_tasks` 限制並行／指派到 slot 上限很緊的專用 pool／檢視查詢模式——改為 batch 或序列化 task。

**👤 Ad-hoc 查詢**

- 目標帳號：個人帳號或團隊存取（如 `da_trading`）。
- 症狀：Cluster loading 飆升、其他 job 變慢或比平常久。
- 根因：在共用的 `bi-warehouse` 上手動跑探索性查詢，影響到 prod job。
- ⚠ 緩解計畫：把這些 ad-hoc 查詢移到 Serverless 以隔離 production（Isaac 正在處理）。

> ℹ **調查流程**：Airflow Alerts dashboard（檢查 scheduled slot 是否飆升）→ AWS Console Query Monitoring（找出出問題的查詢）→ 透過 query metadata 交叉比對 DAG owner。

---

### 第 13 頁 — 可能原因：Airflow Job 被 Skip / Fail

參考用——各原因可能重疊並同時互相觸發。

**🔴 Redshift**

1. Redshift 內部錯誤或服務問題
2. 重度長查詢吃掉 cluster 資源
3. 大量短查詢高並行，耗盡 cluster
4. Leader Node CPU 飆升，拖慢查詢 planning

**🟡 Airflow**

1. `MAX_TIS_PER_QUERY` 太低——scheduler 漏掉低優先級 task
2. 新 DAG 高並行且 `max_active_tasks` 太低——把其他 DAG 資源餓死
3. DAG 的 `priority_weight` 太低 + timeout 太短——永遠來不及被領取

**🔵 連線 / 認證**

1. 目標 DB（Redshift、MySQL 等）故障或維護中
2. 憑證或權限被 GitHub workflow 改動
3. Airflow worker 與目標 cluster 間的網路問題

**🟣 程式碼**

1. 共用通用函式引入 bug
2. 查詢路由機制被更動，破壞了 connection mapping
3. DAG 程式碼變更，對共用資源產生非預期的副作用

> ⚠ 這些原因並非互斥。單一事件（例如部署了一個新的高並行 DAG）可能同時觸發 第 2 欄 → 第 1 欄 → 第 3 欄 的連鎖反應。