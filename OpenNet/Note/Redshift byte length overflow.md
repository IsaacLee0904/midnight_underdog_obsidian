![[Screenshot 2026-04-16 at 12.56.20 PM.png]]
有時候 Airflow 的任務執行失敗會顯示 <mark style="background:#ff4d4f">Load into table `xxxxx` failed. Check `stl_load_errors` system table for details</mark>，這通常是因為 Redshift table 裡面的某個欄位長度不夠容納 raw data 了

Step1. 檢查 stl_load_errors 的紀錄
```SQL

```

