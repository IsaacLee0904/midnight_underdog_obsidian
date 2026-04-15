![[Screenshot 2026-04-13 at 10.07.04 AM.png]]
## Information

* **Jira Ticket** : [Link]([https://opennetltd.atlassian.net/jira/software/projects/BDE/boards/117?jql=assignee%20%3D%20712020%3A4caf52e7-1e32-4e9b-ade5-8c262db7c7f0&selectedIssue=BDE-984](https://opennetltd.atlassian.net/browse/BDE-1147?atlOrigin=eyJpIjoiNjI1Y2M5YjBmZDM1NDJhZGJmNjZhZmE3NTIyYTYyYjgiLCJwIjoiaiJ9))
* **Branch** : <span style="color:rgb(8, 186, 118)">feature/BDE-1147_create_app_account_for_tradingteam</span>
* **Require** : Add a Redshift user `app_metabase_trading` and grant permission for `bi_report.bi_main.src_user_profile_patron_user` and bi_realsports.src_realsports_all_orders_v12` in Sporty Env

![[Screenshot 2026-04-13 at 10.07.04 AM.png]]
## Implement

**Step1. Clone Repo**
Go go [dba-redshift-privileges](https://github.com/opennetltd/dba-redshift-privileges) on Github

**Step2. Create User**
根據需求到 application_account 底下創建 user (`app_metabase_trading`) 並定義
```yaml
type: user
name: app_metabase_trading
env: prod
team: Trading
manager: isaac.lee@opennet.tw
access_rights:
	- cluster_id: sporty-pub-prod-bi-warehouse
	  secret: arn:aws:secretsmanager:eu-central-1:942878658013:secret:redshift!sporty-pub-prod-bi-warehouse-warehouse_admin-2QMALH
	  port: 5439
	  permissions:
		- type: privilege
		  name: role app_metabase_trading
```

**Step3. Grant Permission**
根據需求給予上步驟創建的 user table 權限
```yaml
type: role
name: app_metabase_trading
env: prod
team: Trading
manager: isaac.lee@opennet.tw
access_rights:
	- cluster_id: sporty-pub-prod-bi-warehouse
	  secret: arn:aws:secretsmanager:eu-central-1:942878658013:secret:redshift!sporty-pub-prod-bi-warehouse-warehouse_admin-2QMALH
      port: 5439
      permissions:
		- type: object
		  name: bi_report.bi_main.src_user_profile_patron_user
		  privs: read
		  
		- type: object
		  name: bi_report.bi_realsports.src_realsports_all_orders_v12
		  privs: read
```

**Step4. Raise PR**
發 PR 請 DBA Approve 然後 merge

**Step5. Execute Github Actions**

* Create user
![[Screenshot 2026-04-13 at 5.44.37 PM.png]]

* Grant permission
![[Screenshot 2026-04-13 at 5.46.27 PM.png]]

**Step6. Ask DBA Share the vault**
![[Screenshot 2026-04-13 at 5.56.53 PM.png]]

**Step7. Setup DB Connection in Metabase**
* **Go to Metabase**
	* Prod : https://metabase-ms.on.sportybet2.com/
	* UAT : http://172.31.79.8:3000/
	
* **Setup Connection**
  * Path : Admin Settings > Database > Add database

**Step8. Grant Permission for user**
* **Create a group** 
	* Path : Admin Settings > People > Groups > Create a Group 
	  ![[Screenshot 2026-04-15 at 11.42.35 AM.png]]

* **Add User to the Group**
	* Path : Admin Settings > People > Groups > Click the group >  Add members
	 ![[Screenshot 2026-04-15 at 11.46.33 AM.png]]

* Grant Permission 
	* Path : Admin Settings > Permissions > Databases > 


stanimir marinov