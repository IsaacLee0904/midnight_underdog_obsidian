#### Basic Information
* related DAG
	* <font color="#548dd4">afbet_instant_win.t_instant_win_ticket_info_backfill</font> : original backfill DAGs
	* <font color="#548dd4">afbet_instant_win.t_instant_win_ticket_info_new_backfill</font> : backfill tz and zm data from RDS

Step0. Record the original row count and min(create_time)
* row_count : 
```SQL
-- min(create_time) = 2024-04-01 00:00:03.000
select min(create_time)
from afbet_instant_win_tz.t_instant_win_ticket
where country_code = 'zm'

-- max(create_time) = 2025-06-24 06:01:26.000
select max(create_time)
from afbet_instant_win_tz.t_instant_win_ticket
where country_code = 'zm'

-- 14926664
select count(*)
from afbet_instant_win_tz.t_instant_win_ticket
where country_code = 'zm'
```

Step1. Dual Write + Backfill
PR : https://github.com/opennetltd/warehouse_engineer/pull/2672/changes#diff-d8e39bf16d3763f24dedea0a317d67879473a6b86f80a8b397f449330fd768e7
```markdown
### [BDE-1333](https://opennetltd.atlassian.net/browse/BDE-1333?atlOrigin=eyJpIjoiNWRkNTljNzYxNjVmNDY3MDlhMDU5Y2ZhYzA5YTRkZjUiLCJwIjoiZ2l0aHViLWNvbS1KU1cifQ) Separate TZ and ZM Data for Instant Win Ticket Tables in Redshift

#### Background

Currently, both TZ and ZM data are written into the TZ Redshift table, leaving the ZM table empty. However the ticket main goal is separated those data in Redshift make sure no one will confuse.

#### Workflow

Step1. **Dual Write + Backfill** ← _this PR_

- This tables has been replaced by other table on the application side so only backfill DAG
- Create backfill `_new_backfill` DAG: copy tz and zm historical data from RDS and zm dual write to tz and zm tables
- Validate ZM table data is correct and up-to-date

Step2. **Migrate DA DAGs** :

- Update affected DA DAGs to query from the correct table per country

Step3. **Clean Up** :

- Disable `_new` DAGs and update original DAGs to include both `tz` and `zm` as target countries, each writing only to their respective table
```


Step2. Adjust DA DAGs