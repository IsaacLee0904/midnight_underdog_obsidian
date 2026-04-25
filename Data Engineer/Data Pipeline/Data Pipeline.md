
## Theories of the Data Pipeline

### Speed vs Correctness vs Time (SCT theorem)

類比於 CAP 定理，資料 Pipeline 中同樣存在速度（Speed）、正確性（Correctness）與時間（Time）三者之間的平衡取捨，你可以在速度或正確性之間擇一優先，但無法同時兼顧兩者，確保正確性往往會拖慢 pipeline 的運行速度，因為這涉及全面的稽核與制衡機制，包括資料驗證、錯誤偵測，乃至可能需要人工審查，偏重正確性的取向將延長處理時間，而當速度是首要優先時，這樣的代價往往是難以接受的

![[Pasted image 20260426011353.png]]

### Data Testing vs Data Observability



![[Pasted image 20260426012202.png]]