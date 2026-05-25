tags: #OpenNet  #data-engineering #infra  #airflow #kubernetes  

---
### Background

![[Screenshot 2026-04-02 at 11.36.26 AM.png]]

Rejection pipeline 跑的 Airflow Worker 會在 `de_alert4933` 的 channel 中反覆出現 EKS RAM (>85%) 的 Alarm，起因是因為 Rejection pipeline 有 memory leak 的問題，進到 [Grafana](https://grafana-pub-prod-misc.k8s.on.sportybet2.com/d/adk6avn6n2tc0aee/airflow-eks?orgId=1&from=now-30d&to=now&timezone=utc&var-datasource=P28ADB2B68CA29654&var-airflow_id=airflow-rejection&var-dag_id=$__all&var-airflow_pod=airflow-da-scheduler-658d68cfc7-zzvmg&viewPanel=panel-80)  可以看到這個問題 (如下圖)

![[Screenshot 2026-04-02 at 11.27.04 AM.png]]

### Solution

#### Temporary Solution

![[Rejection pipeline Memory Alert#Solution 定期重啟 Worker]]

#### Improvement Solution

##### What can improve ?
1. <span style="color:rgb(255, 192, 0)">Database connections are never closed after querying with function `get_dataframe_by_sql_without_sharding_db`</span>
* main reason
* 由於 Airflow Worker 是持續運行的 process，且 DAG 每隔幾分鐘就會執行一次，並同時對多個國家的資料庫進行查詢，導致未關閉的連線不斷累積，RAM 持續上升，直到手動重新部署 Worker 才能暫時恢復正常
* table using `get_dataframe_by_sql_withou_sharding_db` function
	* bi_report_rds_conn_ro / bi_report_rds_conn 相關查詢（多數聚合、metabase、alert )
	* 明確使用 _do_not_check_sharding=True 參數的
	* main_{country}_prod_ro
* 其他 function 目前寫法是
```python
_df = pd.read_sql_query(_sql_script, con=_conn)
_conn.close()
```
這會造成一件事，如果 task 中途失敗，就不會正常切斷 connection

2. <font color="#ffc000">Large DataFrames not explicitly deleted after use</font>
* 預防性措施
* 對於這種高頻率、高並發的 pipeline，單純依賴 Python 自動 GC 來回收大型 DataFrame 的記憶體可能不夠及時。在記憶體使用量較高的步驟後加入明確的 gc.collect()呼叫
* [gc docs](https://docs.python.org/3/library/gc.html) 

additional issue : <span style="color:rgb(255, 192, 0)">Intermediate staging files (.feather) are not deleted after use</span>
 * 每次 DAG 執行時，各國家產生的中間 feather 檔案在 combine 步驟完成後並未被清除，這些檔案會持續累積在 Worker 的 ephemeral storage 上，長期下來可能導致 <span style="color:rgb(255, 0, 0)">disk</span> <span style="color:rgb(255, 0, 0)">空間不足</span>，進而造成 pod 被驅逐 (eviction)
 * 有一個 [platform_clean_files](https://airflow-rejection-pub-prod-bi.on.sportybet2.com/dags/platform_clean_files/grid) 的 DAG 理論上會清理乾淨這些 feather 檔案
	 * 清理 export_data & export_data_2512 之下的 feather 跟 csv file

#### Proof of Concept

##### Hypothesis 1 : Connection Leak

<mark style="background:rgba(240, 200, 0, 0.2)">Hypothesis</mark>
In codebase there is a function `DatabaseManager.get_dataframe_by_sql_without_sharding_db` without an explicit `conn.close()`, each call to `get_dataframe_by_sql_without_sharding_db` leaks connection-related resources, causing RSS to grow progressively in long-lived worker processes — increasing process RSS, open file descriptors, and MySQL connected threads.

<mark style="background:rgba(240, 200, 0, 0.2)">Code Fix</mark>
* File : <font color="#548dd4">dags/general_used_functions/rejection_related.py</font>
- Function : `get_dataframe_by_sql_without_sharding_db`
- Was the only DB connection function without explicit `close()`
- Fix : wrapped with `try/finally` to ensure connection is always closed
```python 
_hook, _conn = DatabaseManager.get_hook_and_conn(connection_id, hook_type)
try:
  _df = pd.read_sql_query(sql_script, con=_conn)
finally:
  _conn.close()
```
	
<mark style="background:rgba(240, 200, 0, 0.2)">Benchmark</mark>
- 5000 runs × 2 scenarios (with_close vs without_close)
- Metrics : RSS / Open FD / MySQL Threads_connected
* Script : <font color="#92cddc">dags/rejections/benchmark_connection_leak.py</font>
```bash
airflow dags test --subdir dags/rejections/rejection_pipeline_test.py rejection_pipeline_connection_leak_test 2026-04-16
```

<mark style="background:rgba(240, 200, 0, 0.2)">Result</mark>
![[Pasted image 20260416183023.png]]
Happy path 下，CPython reference counting 會在 function 結束時立刻釋放 connection，有無 `conn.close()` 並無差異，僅在 exception 發生時，traceback 持有 local variable reference，connection 無法被 GC，會造成真正的 connection leak，但這可以透過 `try` and `finally` 解決
 
 **Monitoring**
![[Screenshot 2026-04-20 at 9.49.14 AM.png]]

先把 connection leak 的 fix 推上 Production並經過數天的觀察後發現 memory 持續在增長，因此應該可以排除 connection leak 是主因，查找網路上的資料，發現 Airflow worker 的記憶體持續成長是一個已知問題

[**GitHub Issue #28740 — airflow workers and scheduler memory leak** ](https://github.com/apache/airflow/issues/28740) 
有用戶回報在 Airflow 2.x 上，即使沒有 task 在跑，worker 和 scheduler 的記憶體仍然每天持續增加，process 不會在 task 結束後自動釋放記憶體回到 baseline

[**Medium — What we learned after running Airflow on Kubernetes for 2 years** ](https://medium.com/apache-airflow/what-we-learned-after-running-airflow-on-kubernetes-for-2-years-0537b157acfd) 
文章中提到他們觀察到 worker 記憶體幾乎持續增加，一度懷疑是 task 之間有 memory leak，最終發現關鍵在於兩個 Celery 設定：

- `worker_max_tasks_per_child` — worker process 執行超過 N 個 task 後自動重啟，防止記憶體跨 task 累積
- `worker_max_memory_per_child` — worker process 的記憶體用量超過上限時自動重啟

這兩個設定的核心概念是：**定期回收 worker process，讓 Python allocator 持有的記憶體歸零**，而不是嘗試讓 allocator 自己釋放
##### Hypothesis 2 : CPython pymalloc High-Watermark Effect

<mark style="background:rgba(240, 200, 0, 0.2)">Hypothesis</mark>
After the worker process runs for a long time, RSS keeps growing without returning to baseline. Based on online research, this is a known behavior in long-lived Python processes — CPython's memory allocator may retain freed memory internally rather than returning it to the OS, causing the process RSS floor to rise over time as more tasks are executed.

<mark style="background:rgba(240, 200, 0, 0.2)">Benchmark</mark>
- 3 phases in a single process : small (LIMIT 500) → large (LIMIT 1000) → small (LIMIT 500)
- If high-watermark holds : RSS after Phase 3 should NOT drop back to Phase 1 level
- Metrics : RSS
* Script : <font color="#92cddc">dags/rejections/pymalloc_watermark_test.py</font>
```bash
airflow dags test --subdir dags/rejections/pymalloc_watermark_test.py pymalloc_watermark_test 2026-04-17
```

<mark style="background:rgba(240, 200, 0, 0.2)">Result</mark>
![[Pasted image 20260417144201.png]]

| Phase                                   | RSS Start | RSS End   |
| --------------------------------------- | --------- | --------- |
| Phase 1 — small (LIMIT 500) / 500 runs  | 253.22 MB | 253.35 MB |
| Phase 2 — large (LIMIT 1000) / 500 runs | 253.40 MB | 253.50 MB |
| Phase 3 — small (LIMIT 500) / 1000 runs | 253.50 MB | 253.51 MB |

Phase 3 的起點 = Phase 2 的終點，完全沒有降回 Phase 1 的水位

<mark style="background:rgba(240, 200, 0, 0.2)">Conclusion</mark>
pymalloc high-watermark 是 production RSS 持續上升的機制，Worker process 存活越久，處理的 batch 越多樣，pool 的高水位就持續被推高，RSS 只漲不縮，根本解法是設定 `worker_max_tasks_per_child`，定期重啟 worker process，強制歸零 pymalloc pool
##### Hypothesis 3 : Why Only Rejection Pipeline

<mark style="background:rgba(240, 200, 0, 0.2)">Hypothesis</mark>
所有的 Celery worker 都是一樣的設定，並沒有設定 `worker_max_tasks_per_child` 或 `worker_max_memory_per_child`，代表所有 worker 都不會主動重啟 child process，但只有 rejection worker 出現了明顯的 RSS 增加，假設的原因是溫欸 rejection pipeline 處理的 dataframe 太小，不會觸發 <font color="#ff0000">glibc mmap</font> 路徑，而是走 brk heap，因此永遠不會還給 OS

<mark style="background:rgba(240, 200, 0, 0.2)">Benchmark</mark>
使用從 S3 下載的 prod row count 來生成 feather file，接著執行 `operator_functions_02` 羅輯 (按照國家讀取 feather -> 收集 -> pd.concat -> .copy() -> .groupby() )，同時額外執行一次 `contrast_large` 階段作為對照主，確認 mmap 路徑在同一台機器上確實可以正常觸發，如果假設成立，應該會
* pipeline 個階段 -> `delta_free = 0` 、`hblks` 無變化
* contrast large -> `delta_free < 0`、`hblks` 在第一次執行時有變化

<mark style="background:rgba(240, 200, 0, 0.2)">Result</mark>
Prod 大小的 DataFrame (200 次執行中 `delta_free = 0`並且 `hblks` 一直無變化) 代表沒有觸發 mmap 路徑，而對照組顯示在同一台機器 mmap 是可以正常運作的，因此基本可<font color="#ff0000">以證明 rejection pipeline 的三有記憶體配置都走 brk heap，因而沒有還給 OS</font>

![[result.png]]

#### Redeploy with config
選擇以記憶體為基準的重啟機制 (`WORKER_MAX_MEMORY_PER_CHILD`)，而非以 task 數量為基準(`WORKER_MAX_TASKS_PER_CHILD`)，原因是前者更直接對應 RSS 持續成長的根本問題

門檻設定為 8GB，是因為 HPA 的 scale-out 觸發點約在 9.1GB (worker request 13GB × 70%)，在記憶體到達該觸發點之前，worker child process 會先自動重啟並重置 allocator 狀態，避免不必要的 pod 擴展

### Future Improvements

以下是在調查 BDE-1119 過程中發現的 code-level 改善空間。這些不是記憶體問題的根本原因，但會加速 brk heap 碎片化、拉高 pymalloc 高水位線上升速度。目前 on hold，等 `worker_max_tasks_per_child` config 推上去確認效果後再評估是否處理

1. `pd.concat` in a loop
   在 `operator_functions_02_*` 裡有這種 pattern：
```python
_df_for_all = pd.DataFrame()
for each_country in countries:
    _df = return_df_if_the_feather_file_exists(...)
    _df_for_all = pd.concat([_df_for_all, _df], ignore_index=True)
```
	每次 `pd.concat` 都會產生一個新的完整 DataFrame copy，前一個 `_df_for_all` 要等 GC 
	才釋放，在 loop 過程中記憶體是 O(n²) 的。10 個國家跑完，等於在記憶體裡同時存了 
	1+2+3+...+10 份資料的量

<font color="#ff0000">Fix：collect-then-concat</font>
```python
temp_df_list = []
for each_country in countries:
    _df = return_df_if_the_feather_file_exists(...)
    temp_df_list.append(_df)
_df_for_all = pd.concat(temp_df_list, ignore_index=True)
```

2. `temp_df_list = temp_df_list + [_df]`
	在 `operator_functions_01_*` 和 `operator_functions_02_*` 多處出現：
```python
temp_df_list = []
for each_country in countries:
    temp_df_list = temp_df_list + [_df]  # 每次產生新的 list
```
	用 `+` 而不是 `.append()`，每次迭代都建立新的 list object，舊的 list 要等 GC 回收。

<font color="#ff0000">Fix：`temp_df_list.append(_df)`</font>

3. DataFrame copy 沒有 `del`
```python
_agg_rejection_for_all_df = agg_rejection_df.copy()  # full copy
# ...
agg_rejection_df = pd.concat([agg_rejection_df, _agg_rejection_for_all_df], ...)
# _agg_rejection_for_all_df 沒有被 del，function return 才釋放
```
中間產生的 full copy 在不需要的時候應該主動 `del`，不要等 function scope 結束

4. Module-level `_CONN_CACHE`
```python
_CONN_CACHE: Dict[str, Tuple[float, object]] = {}
_DEFAULT_TTL = 3600
```
Module-level dict 在 worker process 存活期間都存在，entries 只在被存取時才檢查 TTL 是否過期，不會主動清理。不是大問題，但在長時間存活的 worker 裡會慢慢累積 stale entries。可以考慮加定期掃描或用 `cachetools.TTLCache`