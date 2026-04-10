## Information

* **Jira Ticket** : [Link](https://opennetltd.atlassian.net/browse/BDE-1138?atlOrigin=eyJpIjoiMDUwZmMwN2Y4ZjZkNDE0MWIzNmM0NDAzYTVlMTNlMGQiLCJwIjoiaiJ9)
* **Slack Thread** :
	* [original thread](https://opennetltd.slack.com/archives/C07347VTBB2/p1775129308618399)
	* [thread for check with stakeholder](https://opennetltd.slack.com/archives/C054T063YUB/p1775030217492729)
* **Branch** : <span style="color:rgb(8, 186, 118)">feature/BDE-1138_backfill_uat_order_record</span>
![[Screenshot 2026-04-10 at 10.37.59 AM.png]]
* **Detail**：這個任務一開始 Jean 是要求 backfill `za` 的資料，然而在這個資料庫裡面有的資料都已經有同步給他了，後來被 loop 進 stakeholder 的 thread 才發現其實是 `za1`，但 `za1` 不在我們常規 ETL 的範圍內所以要另外加權限給 airflow 使用的帳號密法 `bi_report`

## Implement

#### Step1. Grant permission for bi_report
透過 [dba-application-accounts](https://github.com/opennetltd/dba-application-accounts) 中的 <font color="#00b0f0">app_users/bi_report/uat.yaml</font> 新增權限給 `bi_report`
![[Screenshot 2026-04-10 at 11.19.20 AM.png]]

#### Step2. Create table in Redshift
```SQL
CREATE SCHEMA bi_warehouse.afbet_main_za1;
CREATE TABLE bi_warehouse.afbet_main_za1.t_order_record (LIKE bi_warehouse.afbet_main_za.t_order_record);
```

#### Step3. Adjust the backfill DAG
新增 <font color="#00b0f0">main/afbet_order.t_order_record_shard_backfill.py</font> 跑 backfill 任務