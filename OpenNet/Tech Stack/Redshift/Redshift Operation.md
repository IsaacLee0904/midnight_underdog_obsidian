## Adjust Sort Key

Step1. Check currently Sort key

```sql
SELECT *
FROM svv_table_info
WHERE 1 = 1
	AND schema IN ('afbet_realsports_gh', 'afbet_realsports_ng')
	AND "table" IN ('t_realsports_selection', 't_realsports_selection_cold')
ORDER BY schema;
```

Step2. Change Sort Key

```sql
ALTER TABLE bi_warehouse.afbet_realsports_gh.t_realsports_selection
ALTER SORTKEY (id);
```
- 不需要 rebuild table (但底層還是會重排資料)