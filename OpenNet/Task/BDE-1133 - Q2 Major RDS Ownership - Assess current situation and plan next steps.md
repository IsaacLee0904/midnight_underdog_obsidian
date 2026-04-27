# Background

- 任務背景、目的 - 
- Related Jira or Confluence Link :
	- [Jira Ticket](https://opennetltd.atlassian.net/browse/BDE-1133?atlOrigin=eyJpIjoiYmY0NGE4NDAwNmFkNDBjOGJmZDcxZWZiZjAxNDkzYWYiLCJwIjoiaiJ9)
- Timeline

# Instance Inventory

### Prod Sport

| Alias                     | Endpoint                | Engine       | Instance Type   | Storage | Multi-AZ | Notes                       | AWS console                                                                                                                          |
|:------------------------- | ----------------------- | ------------ | --------------- | ------- | -------- | --------------------------- | ------------------------------------------------------------------------------------------------------------------------------------ |
| bi-report-o1 / bigdata-o1 | sporty-pub-prod-bi-main | Aurora MySQL | db.r6g.12xlarge |         | ✅       | Writer CPU 34%, 18 sessions | [Link](https://eu-central-1.console.aws.amazon.com/rds/home?region=eu-central-1#database:id=sporty-pub-prod-bi-main;is-cluster=true) |
|                           |                         |              |                 |         |          |                             |                                                                                                                                      |


### Prod Encore

### UAT Sport

### UAT Encore

# Performance & Cost

### 每個 instance 的 CPU / Memory / IOPS / connection count

# DAG Review
有哪些 DAG 在寫入或讀取 RDS 的資料？頻率為何？估量每一次的資料量？

# Monitoring Review
