# Background

- 任務背景、目的 - 
- Related Jira or Confluence Link :
	- [Jira Ticket](https://opennetltd.atlassian.net/browse/BDE-1133?atlOrigin=eyJpIjoiYmY0NGE4NDAwNmFkNDBjOGJmZDcxZWZiZjAxNDkzYWYiLCJwIjoiaiJ9)
- Timeline

# Instance Inventory

### Prod Sport

| Alias                     | Endpoint                   | Engine          | Instance Type               | Storage | Multi-AZ | Notes                                             | AWS console                                                                                                                                         |
| :------------------------ | -------------------------- | --------------- | --------------------------- | ------- | -------- | ------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------- |
| bi-report-o1 / bigdata-o1 | sporty-pub-prod-bi-main    | Aurora MySQL    | db.r6g.12xlarge             |         | ✅        | Writer CPU 34%, 18 sessions                       | [Link](https://eu-central-1.console.aws.amazon.com/rds/home?region=eu-central-1#database:id=sporty-pub-prod-bi-main;is-cluster=true)                |
| bi-bigdata-o1             | sporty-pub-prod-bi-bigdata | MySQL Community | db.r6g.xlarge               |         | ✅        | Primary CPU 1.6%, Replica lag 0s                  | [Link](https://eu-central-1.console.aws.amazon.com/rds/home?region=eu-central-1#database:id=sporty-pub-prod-bi-bigdata-instance-1;is-cluster=false) |
| bigdata-ticket-o1         | bigdata-ticket-prod        | Aurora MySQL    | Serverless v2 (40-100 ACUs) |         | ❌        | Writer CPU 51%, Blue/Green deployment in progress | [Link](https://eu-central-1.console.aws.amazon.com/rds/home?region=eu-central-1#database:id=bgd-3kfqd9devzhkx5tb;is-maintenance=true)               |
| metabase-rds              |                            |                 |                             |         |          |                                                   |                                                                                                                                                     |
| bet-bi-o1                 | sporty-global-prod-bet-bi  | Aurora MySQL    | erverless v2 (2-40 ACUs)    |         | ✅        | Writer CPU 1.33%, Reader CPU 3.36%                | [Link](https://eu-central-1.console.aws.amazon.com/rds/home?region=eu-central-1#database:id=sporty-global-prod-bet-bi;is-cluster=true)              |


### Prod Encore

### UAT Sport

### UAT Encore

# Performance & Cost

### 每個 instance 的 CPU / Memory / IOPS / connection count

# DAG Review
有哪些 DAG 在寫入或讀取 RDS 的資料？頻率為何？估量每一次的資料量？

# Monitoring Review
