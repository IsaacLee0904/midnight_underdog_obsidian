
```sql
SELECT
	table_name,
	COUNT(*) AS col_count
FROM information_schema.columns
WHERE table_schema = 'afbet_marketing_shard'
	AND table_name LIKE 't_promotion_gift_%'
	AND column_name = 'early_goals_type'
GROUP BY table_name
ORDER BY table_name;
```