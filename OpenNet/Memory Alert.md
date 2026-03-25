![[Grafana APP 419 PM.png]]

這個 alart 會在 K8s Pod 記憶體使用量過高時觸發，因為可能導致 Pod 重啟並中斷資料管線的運行，因此是一個<span style="color:rgb(255, 0, 0)"><b>高嚴重性很高的 alart</b></span> ，Pod 記憶體使用量過高可能有幾種原因，最常見的情況是 Airflow 任務透過 `pandas` 進行操作導致將資料載入記憶體中

#### What to do ?

<mark style="background: #ABF7F7A6;">Short term solution</mark>
* Find the DAG is causing the OOM issue and stop it
* increase instance size

<mark style="background: #ABF7F7A6;">Long term solution</mark>
* Re-write the DAG to use a more - memory-friendly approach 
	- Export data directly to S3
	- Loading and processing data in multi batches
	* Decrease the size of the data