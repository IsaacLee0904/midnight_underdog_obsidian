![[Screenshot 2026-03-24 at 5.44.07 PM.png]]

這種 alart 表示部分 DAG 執行被跳過或失敗，而 cronjob 會自動重新啟動這些 DAG 任務。
- Airflow 或 Redshift 繁忙時，DAG 可能被跳過或逾時
- DBA 正在對某些資料庫進行維護或升級，可能導致連線失敗
- 某些程式碼邏輯有誤，導致錯誤發生並執行失敗
#### Scenario

<mark style="background: #ABF7F7A6;">Scenario 1 </mark>
![[Al BI Alert APP 1230.png]]
this means a DAG is delayed or behind. backfill job can trigger this and we can mark it as ✅

<mark style="background: #ABF7F7A6;">Scenario 2 </mark>
連續幾次執行失敗，系統有設定一個 task 只會 retry 3 次，失敗 3 次之後不會再重新執行，就需要特別注意並做一些處理
#### What to do ?

**Step1. 實際進去看 DAG 是為什麼錯誤**

**Step2. 看 <mark style="background: #FFB86CA6;">catchup</mark> 設定**
* 如果是 catchup = False ，代表這張表下一次更新的時候是會後蓋前的，只要後面的 run 有成功，就可以直接把失敗的任務改成 Success
* 如果是 catchup = True，代表這張表每一次 run 的資料都是重要的，所以要 trace 為什麼錯誤
	* If the task runs in time but it queued for too long, might need to check the overall Airflow status (ex. [Airflow EKS](https://grafana-pub-prod-misc.k8s.on.sportybet2.com/d/adk6avn6n2tc0aee/airflow-eks?var-datasource=P28ADB2B68CA29654&var-airflow_id=airflow-warehouse&orgId=1&from=now-1h&to=now&timezone=utc&var-dag_id=$__all&var-airflow_pod=) for instance resource usage and Airflow metrics)
	* If the task do have performance issue on query, you can check [Redshift Page](https://eu-central-1.console.aws.amazon.com/redshiftv2/home?region=eu-central-1#/cluster-details?cluster=sporty-pub-prod-bi-warehouse&tab=performance) to see if the cluster is busy. maybe some long running query can be killed to release Redshift resources
	* If both Airflow and Redshift looks normal and no other DAG having issue, probably means you need to modify the DAG code to improve its performance