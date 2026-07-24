

> [!INFO] Scope
> 這份 runbook 針對「Redshift 因 AWS 問題或高負載，導致變慢 / 卡住 / shutdown」的事故情境。
> 分兩部分：**Part 1** 發現問題時怎麼排查、**Part 2** Redshift 回來後怎麼消化 backlog。

## Clusters & Channels

| Cluster | Type | 定位 |
|---|---|---|
| `sporty-pub-prod-bi-warehouse` | Provisioned | 主力 warehouse cluster，warehouse DAG 都跑在這 |
| `data-analysis` | Provisioned | DA 專用 cluster，近期反覆出問題（Leader Node CPU 飆升） |
| `bi-report` | Serverless | 給個人 / 團隊帳號跑查詢用，設有嚴格 timeout |

- **Slack**：`bi_warehouse_alert`（自動告警）、`bi_de`（事故討論 / 通報）
- **On-call**：發生事故時先確認 on-call 有被 tag 到，通報到正確的人

---

# Part 1 — Detecting & Diagnosing an Incident

## 1.1 Alert Sources — 從哪裡發現問題

- **Slack `bi_warehouse_alert`**：DAG fail、`check_redshift_cs` 的 CS 啟用告警、WLM queue 告警
- **Grafana — [Airflow Alerts](https://grafana-pub-prod-misc.k8s.on.sportybet2.com/d/ddvknf88xc3y8d/airflow-alerts?orgId=1&from=now-1h&to=now&timezone=browser)**：看 `Default pool scheduled slots` 是否居高不下（代表 pending task 沒被消化）
- **Grafana — [Airflow DAG (EKS)](https://grafana-pub-prod-misc.k8s.on.sportybet2.com/d/adk6avn6n2tc0aee/airflow-eks?orgId=1&from=now-1h&to=now&timezone=utc&var-airflow_id=airflow-warehouse)**：看 `DAG run schedule delay > 20 mins` 有沒有下降
- **AWS Console — [Redshift Query Monitoring](https://eu-central-1.console.aws.amazon.com/redshiftv2/home?region=eu-central-1#/cluster-details?cluster=sporty-pub-prod-bi-warehouse&tab=queries)**：Leader Node CPU、running / hanging query

## 1.2 Triage — 先分流是哪一類問題

1. **AWS 端故障 / shutdown？** → 查 [AWS Health Dashboard](https://health.aws.amazon.com/health/home)、確認是否已有對 AWS 開的 ticket → 走 Part 2 恢復流程
2. **負載過高（cluster 還活著但很慢）？** → 進 [[#1.3 Load Diagnosis Checklist]]
3. **週二 weekly reboot 造成的預期性 backlog？** → 見 [[Redshift Reboot]]，直接走 Part 2 消化

## 1.3 Load Diagnosis Checklist — cluster 還在但卡住

進 [Redshift Query Monitoring](https://eu-central-1.console.aws.amazon.com/redshiftv2/home?region=eu-central-1#/cluster-details?cluster=sporty-pub-prod-bi-warehouse&tab=queries) 找 hanging / long-running query，並比對以下常見元凶（依過往事故歸納）：

1. 新的 DA job 從很久以前開始持續 **backfill**
2. 新的 DA job **高併發**，同時跑很多查詢
3. 個人 / 團隊帳號（`da_group`、`ds_group`、`da_trading` 等）的 **adhoc 長查詢**
4. **`ANALYZE` / `VACUUM DELETE`** 維護排程正在吃資源
5. **idle session** 卡住，或 **DDL 過多**（例：`src_live_virtuals_summ_min` 每天產生 4000+ DDL）
6. **UNLOAD `PARALLEL OFF`** → 所有匯出資料都經過 leader node，造成 Leader Node CPU 飆升

### Diagnostic Queries

正在跑 / 最近的查詢：
```sql
-- 目前 session 正在跑的查詢
SELECT user_name, db_name, pid, query
FROM stv_recents
WHERE status = 'Running'
ORDER BY pid;
```

長時間 / 卡住的查詢（找出要 kill 的對象）：
```sql
SELECT s.process AS pid,
       s.user_name,
       s.starttime,
       DATEDIFF(minute, s.starttime, GETDATE()) AS running_min,
       q.querytxt
FROM stv_sessions s
LEFT JOIN stl_query q ON s.process = q.pid
WHERE DATEDIFF(minute, s.starttime, GETDATE()) > 10
ORDER BY running_min DESC;
```

Lock / 卡住的 transaction：
```sql
SELECT * FROM svv_transactions WHERE lockable_object_type IS NOT NULL;
```

Table 效能（sort / stats / vacuum 需求）— 詳見 [[Redshift table 查詢出現效能問題時]]：
```sql
SELECT "table", tbl_rows, unsorted, stats_off, vacuum_sort_benefit, skew_rows
FROM svv_table_info
WHERE "schema" LIKE 'afbet_%'
ORDER BY unsorted DESC;
```

## 1.4 Immediate Mitigation — 立即止血

- **Kill 掉** hanging query / adhoc 長查詢 / idle session：
  ```sql
  SELECT pg_terminate_backend(<pid>);
  -- 或
  CANCEL <pid>;
  ```
- 確認 **long-query abort 規則** 有生效：Default queue 任何查詢執行 > 1 小時自動被 kill
- 若元凶是個人帳號的查詢 → 考慮撤銷其 `bi-warehouse` 存取，改導到 `bi-report`(serverless)（見 [dba-redshift-privileges](https://github.com/opennetltd/dba-redshift-privileges)）

---

# Part 2 — Recovery & Backlog Catch-up

Redshift 回來後（或負載退燒後），目標是安全地把積壓的 task 消化掉。

## 2.1 Confirm Recovery — 判斷是否已恢復

- AWS Console：cluster 狀態 `available`、CPU / Leader Node CPU 回落
- WLM queue length 下降（`check_redshift_cs` 會在 queue 連續 10 分鐘 < 3 時自動關 CS）
- Grafana：`Default pool scheduled slots`、`DAG run schedule delay` 指標往下走

## 2.2 Throughput Levers — 增加吞吐的手段

> [!TIP] 原則
> 這些都是**臨時**調整，恢復後要在 [[#2.4 Wrap-up & Restore]] 全部還原。

### A. Enable Concurrency Scaling (CS)
- 自動：`check_redshift_cs` DAG 偵測到 queue 過長會自動 enable，見 [[Redshift Concurrency scaling Alarm]]
- 手動：在 WLM 對應 queue 開啟 CS（`bi-warehouse` / DA queue 都可）
- ⚠️ 恢復後記得關掉，避免額外成本

### B. Shift Workload to Other Cluster
- 把個人 / 團隊帳號的查詢從 `bi-warehouse` 導到 `bi-report`(serverless)
- 同一個 DAG 在 `bi-warehouse` 與 serverless 通常都能正常跑，可暫時分流

### C. Adjust Airflow Pool Slots
- 加 warehouse `default pool` 的 slot 數，讓卡在 `scheduled` 的 task 能被排上
- ⚠️ 已知硬上限約 **120**（設在 Airflow config / EKS，超過設定值也不會生效）
- 卡在 `scheduled` 狀態的 task 是還沒送到 Redshift，通常是 slot 不足 or Redshift 滿了

### D. Skip Resource-Heavy Maintenance Jobs
- 暫時把吃資源的維護 job 標記為 success（跳過）以釋放資源：
  - 每日 `VACUUM DELETE` job
  - `ANALYZE` DAG
- ⚠️ 記得事後補跑或確認下一輪會正常執行

### E. Reboot Cluster（最後手段）
- 當「感覺有東西卡住」且 kill query / 加 slot 都沒改善時才考慮
- Reboot 期間 DAG 會大量 fail，事後走 [[Redshift Reboot]] 的消化流程

## 2.3 Drain the Backlog — 讓積壓追上進度

- 監控 `scheduled slots` / `schedule delay` 持續下降
- 找出正在追進度 / 落後最多的 DAG：
  ```sql
  -- 依 user / DAG 看 query 提交量與耗時，找出壓力來源
  SELECT user_name, COUNT(*) AS query_cnt
  FROM stv_recents
  WHERE status = 'Running'
  GROUP BY user_name
  ORDER BY query_cnt DESC;
  ```
- 優先讓 `high_importance` DAG 追上；backfill 類的可暫緩

## 2.4 Wrap-up & Restore — 收尾

- [ ] 關閉臨時開的 **Concurrency Scaling**
- [ ] 還原 **Airflow pool slots** 到原本設定
- [ ] 恢復被跳過的 **VACUUM DELETE / ANALYZE**（補跑或確認下輪正常）
- [ ] 還原 workload 分流 / WLM / monitoring 的臨時調整
- [ ] Slack 發恢復通知（模板見下）
- [ ] 需要時開 **Incident Report**（含 root cause + action items）

### Slack Recovery Template
```text
^ Redshift weekly reboot with patch version upgrade

^ Redshift is catching up
```

---

## Appendix — Quick Reference

### Kill / Cancel
```sql
SELECT pg_terminate_backend(<pid>);   -- 終止整個 session
CANCEL <pid>;                          -- 只取消當前查詢
```

### Common Root Causes（依過往事故）
1. 新 DA job 長期 backfill
2. 新 DA job 高併發
3. 個人 / 團隊帳號 adhoc 長查詢
4. `ANALYZE` / `VACUUM DELETE` 排程佔資源
5. idle session / 過多 DDL
6. UNLOAD `PARALLEL OFF` 打爆 leader node

### Related Notes
- [[Redshift Reboot]] — 週二 weekly reboot 消化流程
- [[Redshift Concurrency scaling Alarm]] — `check_redshift_cs` 與 CS 自動開關
- [[Redshift table 查詢出現效能問題時]] — `svv_table_info` / ANALYZE / VACUUM
- [[20260619 Redshift Issue]] — 事故討論串（本 runbook 主要來源）

### Key Links
- [AWS Redshift Query Monitoring](https://eu-central-1.console.aws.amazon.com/redshiftv2/home?region=eu-central-1#/cluster-details?cluster=sporty-pub-prod-bi-warehouse&tab=queries)
- [Airflow Alerts (Grafana)](https://grafana-pub-prod-misc.k8s.on.sportybet2.com/d/ddvknf88xc3y8d/airflow-alerts?orgId=1&from=now-1h&to=now&timezone=browser)
- [Airflow DAG / EKS (Grafana)](https://grafana-pub-prod-misc.k8s.on.sportybet2.com/d/adk6avn6n2tc0aee/airflow-eks?orgId=1&from=now-1h&to=now&timezone=utc&var-airflow_id=airflow-warehouse)
- [dba-redshift-privileges](https://github.com/opennetltd/dba-redshift-privileges)
