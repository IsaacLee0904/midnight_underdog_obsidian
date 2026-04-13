![[Screenshot 2026-04-13 at 10.07.04 AM.png]]
## Information

* **Jira Ticket** : [Link]([https://opennetltd.atlassian.net/jira/software/projects/BDE/boards/117?jql=assignee%20%3D%20712020%3A4caf52e7-1e32-4e9b-ade5-8c262db7c7f0&selectedIssue=BDE-984](https://opennetltd.atlassian.net/browse/BDE-1147?atlOrigin=eyJpIjoiNjI1Y2M5YjBmZDM1NDJhZGJmNjZhZmE3NTIyYTYyYjgiLCJwIjoiaiJ9))
* **Branch** : <span style="color:rgb(8, 186, 118)">feature/BDE-1147_create_app_account_for_tradingteam</span>
* **Require** : Add a Redshift user `app_metabase_trading` and grant permission for `bi_report.bi_main.src_user_profile_patron_user` and bi_realsports.src_realsports_all_orders_v12` in Sporty Env

![[Screenshot 2026-04-13 at 10.07.04 AM.png]]
## Implement

**Step1. Clone Repo**
Go go [dba-redshift-privileges](https://github.com/opennetltd/dba-redshift-privileges) on Github

Step2. Create User
根據需求到 application_account 底下創建 user 並定義
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

Step3. 