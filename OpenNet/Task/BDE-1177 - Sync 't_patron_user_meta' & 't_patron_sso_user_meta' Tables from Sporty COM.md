![[Screenshot 2026-04-16 at 3.42.52 PM.png]]

`t_patron_user_meta`
- There was a exist pipeline (`sporty_patron.t_patron_user_meta_backfill.py`) will use it for backfill.
- target table : `sporty_patron_global.t_patron_user_meta``
- backfill start : min(create_time)
- dag_schedule : daily
- min(create_time) : 2020-09-11 10:02:56.504
- 20260416 11:14 start to backfill, ETA 20260418
- row_count : 9610323
    
`t_patron_sso_user_meta`
- target table : `sporty_patron_global.t_patron_sso_user_meta`
- backfill start : min(create_time)
- dag_schedule : daily
- min(create_time) : 2021-12-02 03:06:07.460150
- row_count : 23964481

其中 `t_patron_user_meta` 是之前有用 UTS  backfill 過的，做法是去 Redshift 裡面的 `bi_wh_monitoring.uts_backfill_state` UPDATE 那筆紀錄讓他從更久以前開始回跑
```SQL
-- Check the exist record
SELECT * FROM bi_wh_monitoring.uts_backfill_state
WHERE module = 'sporty_patron'
AND source_schema = 'sporty_patron'
AND table_name = 't_patron_sso_user_meta';

-- Update backfill record
UPDATE bi_wh_monitoring.uts_backfill_state
SET cursor_start_ts = '2020-09-11T10:02:56', -- will backfill from this time
update_time = GETDATE()
WHERE brand = 'sporty'
	AND env = 'prod'
	AND module = 'sporty_patron'
	AND source_schema = 'sporty_patron'
	AND table_name = 't_patron_user_meta';
```

然後因為回跑的時間區間很長並且很急迫所以可以透果調整 DAG 來做
```python
backfill_schedule = "*/5 * * * *" # run every 5 mins

start_dt_conf = (
	datetime.strptime(context["dag_run"].conf.get("start_time"), "%Y-%m-%dT%H:%M:%S")
	if context.get("dag_run") and context["dag_run"].conf and context["dag_run"].conf.get("start_time")
	else start_dt
)

end_dt_conf = (
	datetime.strptime(context["dag_run"].conf.get("end_time"), "%Y-%m-%dT%H:%M:%S")
	if context.get("dag_run") and context["dag_run"].conf and context["dag_run"].conf.get("end_time")
	else (start_dt_conf + timedelta(days=90)) # 調整成 90 天，一次跑 90 天資料
)
```