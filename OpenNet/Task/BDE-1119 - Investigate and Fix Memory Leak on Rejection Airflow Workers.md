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
		  
2. <span style="color:rgb(255, 192, 0)">Intermediate staging files (.feather) are not deleted after use</span>
 * 每次 DAG 執行時，各國家產生的中間 feather 檔案在 combine 步驟完成後並未被清除，這些檔案會持續累積在 Worker 的 ephemeral storage 上，長期下來可能導致 <span style="color:rgb(255, 0, 0)">disk</span> <span style="color:rgb(255, 0, 0)">空間不足</span>，進而造成 pod 被驅逐 (eviction)

3. <span style="color:rgb(255, 192, 0)">No GC trigger between tasks</span>
* 預防性措施
* 對於這種高頻率、高並發的 pipeline，單純依賴 Python 自動 GC 來回收大型 DataFrame 的記憶體可能不夠及時。在記憶體使用量較高的步驟後加入明確的 gc.collect()呼叫，是一個防禦性的改善措施，有助於降低記憶體的峰值使用量
* [gc docs](https://docs.python.org/3/library/gc.html) 