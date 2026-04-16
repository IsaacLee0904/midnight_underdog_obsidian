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

#### Proof of Concept

##### Hypothesis 1 : Connection Leak

<mark style="background:rgba(240, 200, 0, 0.2)">Hypothesis</mark>
In `DatabaseManager.get_dataframe_by_sql_without_sharding_db`, if connection is not closed, long-lived worker processes will accumulate connection-related resources across
repeated pipeline runs, causing :
  - increasing process RSS
  - increasing open file descriptors (FD)
  - increasing MySQL connected threads

<mark style="background:rgba(240, 200, 0, 0.2)">Code Findings</mark>

<font color="#c3d69b">1) High-risk function</font>
- File: `dags/general_used_functions/rejection_related.py`
- Function: `get_dataframe_by_sql_without_sharding_db(...)`
 * Fix applied
```python 
_hook, _conn = DatabaseManager.get_hook_and_conn(connection_id, hook_type)
try:
  _df = pd.read_sql_query(sql_script, con=_conn)
finally:
  _conn.close()
```

<span style="color:rgb(255, 0, 0)"><font color="#c3d69b">2) Why this matters</font></span>
* This function is frequently used by rejection pipeline tasks (especially non-sharded and BI read/write paths).
* Is the only db connect function without <font color="#92cddc">close()</font>
	
<mark style="background:rgba(240, 200, 0, 0.2)">Benchmark (Causal Validation)</mark>
-> close vs no-close

Environment
* Docker compose with :
	* airflow container
	* mysql:8.0 container
* Script :
	* <font color="#92cddc">dags/rejections/benchmark_connection_leak.py</font>
```bash
airflow dags test --subdir dags/rejections/rejection_pipeline_test.py rejection_pipeline_connection_leak_test 2026-04-16
```

Metics captured at each run-end baseline
1. Process RSS (MB)
2. Open FDs
3. MySQL Threads_connected

Scenarios
1. FIX : close connection every query
2. BUG : no close + leak emulation to mimic long-lived retained resources

Result
![[Pasted image 20260416172604.png]]



 monitoring
```bash
while true; do                                                               
    docker stats airflow-benchmark --no-stream --format '{{.MemUsage}}'
    sleep 1                                                                    
done
```







[What we learned after running Airflow on Kubernetes for 2 years](https://medium.com/apache-airflow/what-we-learned-after-running-airflow-on-kubernetes-for-2-years-0537b157acfd)
[Github issue : airflow workers and scheduler memory leak](https://github.com/apache/airflow/issues/28740)
