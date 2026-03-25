![[Screenshot 2026-03-24 at 5.13.42 PM.png]]

我們有一個 [Airflow 排程](https://airflow-warehouse-pub-prod-bi.on.sportybet2.com/dags/table_update_check_all/grid?search=table_update_check_all)會檢查 `public.table_update_check` 這張表，看哪些 table 沒有依照他應該更新的頻率更新，並出現上圖的 alarm

**Step1. 進到 code base 看這個 table 是由哪一個 DAG 更新的**

**Step2. 然後看這個 DAG 是為什麼沒有更新**

**Step3. 解決問題**
* 如果 DAG 真的有問題就看 Airflow log 錯誤在哪，怎麼解決
* 如果他沒寫入是正常的，那就去 table_update_check 刪掉那個 table 紀錄
```sql
DELETE
FROM public.table_update_check
where table_id = '`connections_versioning`'
```
