# Background

- 任務背景、目的 - 
- Related Jira or Confluence Link :
	- [Jira Ticket](https://opennetltd.atlassian.net/browse/BDE-1133?atlOrigin=eyJpIjoiYmY0NGE4NDAwNmFkNDBjOGJmZDcxZWZiZjAxNDkzYWYiLCJwIjoiaiJ9)
- Timeline

# Instance Inventory

### Prod Sport

```dataview
TABLE alias, engine, instance_type, cpu_writer, multi_az, environment FROM "BDE-1133" SORT environment ASC
```



### Prod Encore

### UAT Sport

### UAT Encore

# Performance & Cost

### 每個 instance 的 CPU / Memory / IOPS / connection count

# DAG Review
有哪些 DAG 在寫入或讀取 RDS 的資料？頻率為何？估量每一次的資料量？

# Monitoring Review
