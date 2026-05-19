## Information

* **Jira Ticket** : [Link](https://opennetltd.atlassian.net/browse/BDE-1259?atlOrigin=eyJpIjoiOWQ3Y2E5MGM2ODU1NGQ5Y2EzNWJkMzcyNTFmNDFjZWQiLCJwIjoiaiJ9)
* **Branch** : <span style="color:rgb(8, 186, 118)">feature/BDE-1259-sync_blog_rewrites_v2_to_be_rds</span>
* **Require** : 將 AI re-write 的文章回寫到 application 的 DB

![[Screenshot 2026-05-11 at 2.20.57 PM.png]]

### Step1. Create DAG

For this kind of reverse ETL we refer to  [data_analysis](https://github.com/opennetltd/data_analysis) repo.  Use `load_into_rds_from_s3_batch` in the DAG to avoid High DML Latency alaert caused by loading S3 files to RDS. See [t_pocket_user_lifetime_bet](https://airflow-da-pub-prod-bi.on.sportybet2.com/dags/t_pocket_user_lifetime_bet/grid) as a reference.

![[Screenshot 2026-05-14 at 5.44.36 PM.png]]
### Step2. Create Application Database

Reach out <font color="#548dd4">Kenneth</font> for assistance. Jira ticket must be submitted to the DBA team following the
 [CREATE New Table Structure (Simplified version) (SOP)](https://opennetltd.atlassian.net/wiki/spaces/DBA/pages/4252532749/CREATE+New+Table+Structure+Simplified+version+SOP) including : 

1. Scope Information
2. Archive Rule
3. Table Definition 
4. Capacity Assessment
5. Query Usage Examples

Refere to [DBA-11492](https://opennetltd.atlassian.net/browse/DBA-11492?atlOrigin=eyJpIjoiNWZmYzJhZTQxMDRmNGE2NjkxZGEwYmQ2YWE3ZjRhYTMiLCJwIjoiaiJ9) as an example. Note that approval from the Backend team who owns the database is required before proceeding.
![[Screenshot 2026-05-14 at 4.45.18 PM.png]]

### Step3. Create Airflow Connection or Grant Permission

After the tables are created, grant the necessary permissions to the DB users used by our Airflow DAGs. Existing Airflow MySQL accounts are stored in the 1Password vault <font color="#ffc000">Airflow-connections</font>.

![[Screenshot 2026-05-14 at 6.00.07 PM.png]]

For the full list of MySQL accounts, refer to the table below : 

| account name         | note                                                                                         |
| -------------------- | -------------------------------------------------------------------------------------------- |
| app_airflow_ro       | SELECT only. Use for reading any source DB (primaries, replicas, games, risk, recomm, etc.). |
| app_airflow_rw       | CRUD. Use for BI/ETL destination schema writes to Sporty/Encore primaries.                   |
| app_da_airflow_rw    | CRUD. Use for DA writes (afbet_bi, afbet_recap, sporty_rm_bi, afbet_realsports table-level). |
| app_de_airflow_rw    | CRUD. Use for DE writes (sporty_rm schema).                                                  |
| app_games_airflow_rw | CRUD. Use for Games writes (afbet_lobby, afbet_sporty_hero schemas).                         |

**Step3-1. Grant Permission to the User**
To grant permissions, edit the yaml file in <font color="#548dd4">/app_users/{account_name}</font> within this [repo](https://github.com/opennetltd/dba-application-accounts) then open a PR and request DBA (<font color="#548dd4">Kenneth</font>) approval, the workflow can be trigged.

![[Screenshot 2026-05-14 at 5.37.09 PM.png]]

**Step3-2. Create a New Airflow Connection**
If the connection doesn't exist in the vault, a new one will need to be created. Follow [Airflow MySQL Connections – AWS Secrets Manager Guide](https://opennetltd.atlassian.net/wiki/spaces/DET/pages/4463263787/Airflow+MySQL+Connections+AWS+Secrets+Manager+Guide?atlOrigin=eyJpIjoiY2RjNjQ0NDkyMGY1NGUyMmE0OTRlNTQzZmQwNDM0YTEiLCJwIjoiY29uZmx1ZW5jZS1jaGF0cy1pbnQifQ#Requesting-a-New-Connection), or reach out to <font color="#548dd4">Ken</font> for assistance. The DevOps naming in 1Password is the connection ID that should be used in the DAG. See the screenshot below for reference:
![[Screenshot 2026-05-14 at 6.09.24 PM.png]]
![[Screenshot 2026-05-14 at 6.07.13 PM.png]]

### Step4. Associate IAM Role with Redshift Cluster

When running the DAG, you may encounter some error

#### 1. Redshift missing IAM role to dump data to S3

![[Pasted image 20260515112014.png]]

This occurs when the Redshift cluster does not have the required IAM role attached, which is needed to allow Redshift to dump data to S3. In this case, please open a ticket and reach out to DevOps (<font color="#548dd4">Robin</font>) to associate the IAM role.

#### 2. RDS missing IAM role to load data from S3

![[Pasted image 20260519114534.png]]

This occurs when the RDS cluster does not have the required IAM role attached, which is needed to allow load data from S3. In this case, please ask the DBA who own the database to associate the IAM role.
![[Screenshot 2026-05-19 at 1.50.58 PM.png]]