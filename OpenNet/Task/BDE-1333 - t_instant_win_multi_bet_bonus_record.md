#### Basic Information
* related DAG
	* <font color="#548dd4">afbet_instant_win.t_instant_win_multi_bet_bonus_record</font> & <font color="#548dd4">afbet_instant_win.t_instant_win_multi_bet_bonus_record_new</font> : DAG for sync data to warehouse
	* <font color="#548dd4">afbet_instant_win.t_instant_win_league_backfill</font> : copy data from tz to zm

Step0. Record the original row count and min(create_time)
* row_count : 296
```SQL
-- min(create_time) = 2025-01-29 00:00:00.000
select min(create_time)
from afbet_instant_win_tz.t_instant_win_multi_bet_bonus_record

-- RDS : 9455410

select count(*)
from afbet_instant_win.t_instant_win_multi_bet_bonus_record tiwmbbr
where create_time >= '2025-01-01 00:00:00.000'
```

Step1. Dual Write + Backfill
PR : https://github.com/opennetltd/warehouse_engineer/pull/2702
```markdown
### [BDE-1333] Separate TZ and ZM Data for Instant Win Tables in Redshift

#### Background

Currently, both TZ and ZM data are written into the TZ Redshift table, leaving the ZM table empty. However the ticket main goal is separated those data in Redshift make sure no one will confuse.

#### Workflow

Step1. **Dual Write + Backfill** ← _this PR_

- Update DAGs : ZM data writes to both `afbet_instant_win_tz.t_instant_win_market_pool` (existing, for backward compatibility) and `afbet_instant_win_zm.t_instant_win_market_pool` (new target), this pipeline original don't have `_new` to write zm data to tz.
- Create `_backfill` DAG: full load backfill historical records to ensure ZM table is complete before DA migration
- Validate ZM table data is correct and up-to-date

Step2. **Migrate DA DAGs** :

- Update affected DA DAGs to query from the correct table per country

Step3. **Clean Up** :

- Disable `_new` DAGs and update original DAGs to include both `tz` and `zm` as target countries, each writing only to their respective table
```


Step2. Adjust DA DAGs