o![[Screenshot 2026-03-30 at 11.25.22 AM.png]]

### Create Table
因為這個需求單有特別說到需要以 `user_id` 作為 distkey，因此需要另外使用 dba_tools 來手動建立 table

>[!important] `__all_countires__` 
>* 在 Prod 包含：`br` , `gh` , `int`, `ke`, `ng`, `tz`, `za`, `zm` 8 個國家
>* 在 UAT 包含：`gh`, `int`, `ng`, `tz`, `za` 5 個國家

**Step1. 創建 SQL file**
在 [dba_redshfit_executor](https://github.com/opennetltd/dba-redshift-executor) & [dba_redshift_executor_prod](https://github.com/opennetltd/dba-redshift-executor-prod) 兩個 repo 下的 ddl folder 創建建立 table 的 SQL file

**Step2. 等待 Approve and Merge**
![[Screenshot 2026-03-30 at 2.31.02 PM.png]]
到 Slack 發訊息請別人 Approve 之後 Merge

**Step3. Run SQL on Redshift**
![[Screenshot 2026-03-30 at 2.33.49 PM.png]]

### Create Table


#### Reference
[[Unified Table Sync (UTS)]]