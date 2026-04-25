
## Theories of the Data Pipeline

### Speed vs Correctness vs Time (SCT theorem)

類比於 CAP 定理，資料 Pipeline 中同樣存在速度（Speed）、正確性（Correctness）與時間（Time）三者之間的平衡取捨，你可以在速度或正確性之間擇一優先，但無法同時兼顧兩者，確保正確性往往會拖慢 pipeline 的運行速度，因為這涉及全面的稽核與制衡機制，包括資料驗證、錯誤偵測，乃至可能需要人工審查，偏重正確性的取向將延長處理時間，而當速度是首要優先時，這樣的代價往往是難以接受的

![[Pasted image 20260426011353.png]]

### Data Testing vs Data Observability

<font color="#ff0000">資料測試</font>（Data Testing）與<font color="#ff0000">資料可觀測性</font>（Data Observability）是資料品質的兩個重要面向，資料測試確保資料符合特定的要求與規範；資料可觀測性則是對資料進行持續監控，以識別問題並協助排查，
這兩者是相輔相成的資料品質實踐方法：資料測試著重於防範問題的發生，而資料可觀測性則著重於識別並解決已發生的問題

![[Pasted image 20260426012202.png]]