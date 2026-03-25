![[Screenshot 2026-03-19 at 11.37.41 AM.png]]

這個 alart 會檢查 Redshift 的系統資料表，尋找執行時間過長的查詢。這有助於找出可能需要手動終止的 hanging 或 zombie query

#### What to do ?
通常是 Redshift 的高負載導致的，到 [Redshift Query monitoring](https://eu-central-1.console.aws.amazon.com/redshiftv2/home?region=eu-central-1#/cluster-details?cluster=sporty-pub-prod-bi-warehouse&tab=queries) to check if there is hanging query or user trigger long query need to be killed.