目前有自定義一些 function EX. _run_sql_in_redshift 來讓 DA 去做使用做到可以切分成不同的 queue 執行才能確保 workload management，但目前沒有足夠的機制在檢查他們是不是有使用這些 function

1. CICD pipeline 裡面強制檢查
2. 類似 dbt document coverage 工具去掃，看哪些 DAG 沒有使用這些 function