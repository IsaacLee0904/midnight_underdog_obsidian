#### Basic Information
* related DAG
	* <font color="#548dd4">afbet_instant_win.t_instant_win_user_round_settled_pt</font> & <font color="#548dd4">afbet_instant_win.t_instant_win_user_round_settled_pt_new</font> : DAG for sync data to warehouse
	* <font color="#548dd4">afbet_instant_win.t_instant_win_user_round_settled_pt_zm_backfill</font> : backfill DAG for backfill zm data 

Step0. Record the original row count and min(create_time)
* row_count : 
```SQL
-- min(create_time) = 2025-01-01 00:00:12.000 >> backfill should from 2025-01-01 00:00:00
select min(create_time)
from afbet_instant_win_tz.t_instant_win_user_round_settled_pt

-- RDS : 2162489
select count(*)
from afbet_instant_win.t_instant_win_user_round_settled_pt
where create_time >= '2021-01-01 00:00:00'
```

Step1. Dual Write + Backfill
PR : https://github.com/opennetltd/warehouse_engineer/pull/2804
```markdown
### [BDE-1333] Separate TZ and ZM Data for Instant Win Ticket Tables in Redshift

#### Background

Currently, both TZ and ZM data are written into the TZ Redshift table, leaving the ZM table empty. However the ticket main goal is separated those data in Redshift make sure no one will confuse.

#### Workflow

Step1. **Dual Write + Backfill** ← _this PR_

- Create `_new` DAGs : ZM data writes to both `afbet_instant_win_tz.t_instant_win_user_round_settled_pt` (existing, for backward compatibility) and `afbet_instant_win_zm.t_instant_win_user_round_settled_pt` (new target)
- Create `_zm_backfill` DAG: backfill historical records to ensure ZM table is complete before DA migration
- Validate ZM table data is correct and up-to-date

Step2. **Migrate DA DAGs** :

- Update affected DA DAGs to query from the correct table per country

Step3. **Clean Up** :

- Disable `_new` DAGs and update original DAGs to include both `tz` and `zm` as target countries, each writing only to their respective table
```


Step2. Adjust DA DAGs