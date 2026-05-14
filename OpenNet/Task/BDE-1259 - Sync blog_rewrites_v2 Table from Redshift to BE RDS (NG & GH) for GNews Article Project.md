## Information

* **Jira Ticket** : [Link](https://opennetltd.atlassian.net/browse/BDE-1259?atlOrigin=eyJpIjoiOWQ3Y2E5MGM2ODU1NGQ5Y2EzNWJkMzcyNTFmNDFjZWQiLCJwIjoiaiJ9)
* **Branch** : <span style="color:rgb(8, 186, 118)">feature/BDE-1259-sync_blog_rewrites_v2_to_be_rds</span>
* **Require** : 將 AI re-write 的文章回寫到 application 的 DB

![[Screenshot 2026-05-11 at 2.20.57 PM.png]]

### Step1. Create DAG

For this kind of reverse ETL we keep in [data_analysis](https://github.com/opennetltd/data_analysis) repo. Could reference [t_pocket_user_lifetime_bet](https://airflow-da-pub-prod-bi.on.sportybet2.com/dags/t_pocket_user_lifetime_bet/grid) and we used to occur a High DML Latency alert caused by loading too many S3 files data into RDS trigger by reverse ETL. Make sure using l`oad_into_rds_from_s3_batch` function in the DAG
### Step2. Create Application Database

Could ask <font color="#548dd4">Kennth</font> for help and need to provide original Jira ticket for them.
Create a Jira ticket for DBA following the format in [CREATE New Table Structure (Simplified version) (SOP)](https://opennetltd.atlassian.net/wiki/spaces/DBA/pages/4252532749/CREATE+New+Table+Structure+Simplified+version+SOP) include information below :

1. Scope Information
2. Archive Rule
3. Table Definition 
4. Capacity Assessment
5. Query Usage Examples

ref : [DBA-11492](https://opennetltd.atlassian.net/browse/DBA-11492?atlOrigin=eyJpIjoiNWZmYzJhZTQxMDRmNGE2NjkxZGEwYmQ2YWE3ZjRhYTMiLCJwIjoiaiJ9)

In this step, would also need discuss and get approval from Backend who own the database 
![[Screenshot 2026-05-14 at 4.45.18 PM.png]]

### Step3. Grant Permission to MySQL Account

After DBA help create those tables, need to grant permission to the db user ( for all the MySQL account please see the table below ) which we running our Airflow DAG please follow the [Confluence page](https://opennetltd.atlassian.net/wiki/spaces/DET/pages/4463263787/Airflow+MySQL+Connections+AWS+Secrets+Manager+Guide?atlOrigin=eyJpIjoiY2RjNjQ0NDkyMGY1NGUyMmE0OTRlNTQzZmQwNDM0YTEiLCJwIjoiY29uZmx1ZW5jZS1jaGF0cy1pbnQifQ#Requesting-a-New-Connection) or ask <font color="#548dd4">Ken</font> for help. The exist airflow users are list in 1Password vault <font color="#ffc000">Airflow-connections</font>

| account name         | note                                                                                         |
| -------------------- | -------------------------------------------------------------------------------------------- |
| app_airflow_ro       | SELECT only. Use for reading any source DB (primaries, replicas, games, risk, recomm, etc.). |
| app_airflow_rw       | CRUD. Use for BI/ETL destination schema writes to Sporty/Encore primaries.                   |
| app_da_airflow_rw    | CRUD. Use for DA writes (afbet_bi, afbet_recap, sporty_rm_bi, afbet_realsports table-level). |
| app_de_airflow_rw    | CRUD. Use for DE writes (sporty_rm schema).                                                  |
| app_games_airflow_rw | CRUD. Use for Games writes (afbet_lobby, afbet_sporty_hero schemas).                         |


Step3-1. Grant permission to the user
Grant permission for the right database user with this [repo](https://github.com/opennetltd/dba-application-accounts) . Edit the yaml file in <font color="#548dd4">/app_users/{account_name}</font> and ask DBA to approve the PR. Then could run the workflow.

![[Screenshot 2026-05-14 at 5.37.09 PM.png]]

Step2-2. 

