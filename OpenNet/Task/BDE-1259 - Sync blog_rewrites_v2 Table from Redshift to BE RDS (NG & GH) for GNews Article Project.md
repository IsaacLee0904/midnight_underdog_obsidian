## Information

* **Jira Ticket** : [Link](https://opennetltd.atlassian.net/browse/BDE-1259?atlOrigin=eyJpIjoiOWQ3Y2E5MGM2ODU1NGQ5Y2EzNWJkMzcyNTFmNDFjZWQiLCJwIjoiaiJ9)
* **Branch** : <span style="color:rgb(8, 186, 118)">feature/BDE-1259-sync_blog_rewrites_v2_to_be_rds</span>
* **Require** : 將 AI re-write 的文章回寫到 application 的 DB

![[Screenshot 2026-05-11 at 2.20.57 PM.png]]

Sometimes we would need to 
### Step1. Create Ticket for DBA

Usually ask <font color="#548dd4">Kennth</font> for help would need to provide original Jira ticket for them.
And create a Jira ticket for DBA following the format in [CREATE New Table Structure (Simplified version) (SOP)](https://opennetltd.atlassian.net/wiki/spaces/DBA/pages/4252532749/CREATE+New+Table+Structure+Simplified+version+SOP) include information below :

1. Scope Information
2. Archive Rule
3. Table Definition 
4. Capacity Assessment
5. Query Usage Examples

After create ticket will need the related Backend to approve this ticket let BDA move on to next step.

### Step2. Grant Permission for Airflow user

While DBA help create those tables, need to grant permission to the app_airflow user which we running our Airflow DAG
