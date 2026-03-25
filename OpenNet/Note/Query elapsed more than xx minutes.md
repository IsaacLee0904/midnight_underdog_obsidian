![[Screenshot 2026-03-19 at 11.37.41 AM.png]]

這個 alart 會檢查 Redshift 的系統資料表，尋找執行時間過長的查詢。這有助於找出可能需要手動終止的 hanging 或 zombie query

#### What to do ?
通常是 Redshift 的高負載導致的，到 [Redshift Query monitoring](https://eu-central-1.console.aws.amazon.com/redshiftv2/home?region=eu-central-1#/cluster-details?cluster=sporty-pub-prod-bi-warehouse&tab=queries) 檢查是不是有 hanging or zombie query

**Step1. Connect to database**
![[Pasted image 20260325151544.png]]
- Create a new connection
- Database name : <span style="color:rgb(8, 186, 118)">bi_warehouse</span> /  Database user : <span style="color:rgb(8, 186, 118)">bi_report</span> 

**Step2. go to AWS Redshift [Query Monitoring page](https://eu-central-1.console.aws.amazon.com/redshiftv2/home?region=eu-central-1#/cluster-details?cluster=sporty-pub-prod-bi-warehouse&tab=queries) to check for running query**
![[Screenshot 2026-03-19 at 11.39.15 AM.png]]

**Step3. if the query should be stopped, click the query link to the Query page. and click Terminate Query to kill the running query**
![[Screenshot 2026-03-19 at 11.39.43 AM.png]]

