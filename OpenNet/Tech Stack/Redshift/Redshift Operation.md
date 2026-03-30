## Check Table info
### with data
```sql
SHOW search_path;

SET search_path to afbet_marketing_br -- need to assign a schema

SELECT *
FROM svv_table_info
WHERE 1 = 1
	AND schema IN ('afbet_realsports_gh', 'afbet_realsports_ng')
	AND "table" IN ('t_realsports_selection', 't_realsports_selection_cold')
ORDER BY schema;
```

#### without data
```sql
select *
from pg_table_def
where schemaname = 'afbet_marketing_br' and tablename = 't_promotion_odds_boost_lfb_log'
```

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