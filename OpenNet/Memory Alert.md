![[Grafana APP 419 PM.png]]

這個 alart 會在 K8s Pod 記憶體使用量過高時觸發，因為可能導致 Pod 重啟並中斷資料管線的運行，因此是一個<span style="color:rgb(255, 0, 0)"><b>高嚴重性很高的 alart</b></span> ，Pod 記憶體使用量過高可能有幾種原因，最常見的情況是 Airflow 任務透過 `pandas` 進行操作導致將資料載入記憶體中

#### What to do ?

Short term solution

Long term solution