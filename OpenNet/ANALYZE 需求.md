![[Screenshot 2026-03-24 at 4.22.11 PM.png]]


**Step1. 先看他的 Query Plan** 
如果是`DS_DIST_NONE` 或 `DS_DIST_ALL_NONE` 代表 `JOIN` 沒什麼問題都有使用到 distribution key，但如果是其他種代表他沒有正確的使用到 distribution key 效能必較差是可以預期的，可能要請他注意一下

>According to the query plan, the problem is on join trigger `DS_DIST_ALL_INNER`, can you check the cols using on `JOIN` ?

**Step2. 幫他跑 ANALYZE** 
* 先下 query 看能優化多少
```sql
SELECT * 
FROM svv_table_info
WHERE "schema" LIKE 'afbet_realsports_%'
AND "table" IN ('t_realsports_selection');
```
![[Screenshot 2026-03-24 at 5.11.15 PM.png]]

* 下 query 去優化
```sql 
ANALYZE afbet_pocket_gh.t_pocket_pay_record_ch_request;
```

#### Reference
[[Redshift table 查詢出現效能問題時]]