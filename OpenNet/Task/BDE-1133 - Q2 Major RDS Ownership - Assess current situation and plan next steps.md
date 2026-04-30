# Overview
### Background

BI RDS instances have historically been managed by the DBA team, with ad-hoc support from the DA and DE teams when issues arise. As the data infrastructure grows, this arrangement has created gaps in ownership — particularly around proactive monitoring, cost visibility, and scalability planning.

To address this, the [DE team is taking formal ownership of these RDS instances](https://opennetltd.atlassian.net/browse/BDE-1133?atlOrigin=eyJpIjoiZTcxMTJjMjliYmFhNDIyNGE1Y2RiMzljOWRmMGU3YzciLCJwIjoiaiJ9) going forward.

### Objective

This document aims to：
- Establish a clear inventory of all BI RDS instances (Prod and UAT)
- Assess the current performance and cost of each instance
- Identify existing issues and risks that require immediate or near-term action
- Review the DAGs connected to these instances and understand their dependencies
- Evaluate the current monitoring and alerting coverage
- Deliver a prioritized action plan for the DE team to act on

### Scope

This assessment covers BI RDS instances only, excluding service backends such as Airflow and Metabase RDS.

![[Screenshot 2026-04-30 at 11.15.56 AM.png]]

### Backlog
```txt
1. Overview
   - 背景與目標

2. Instance Inventory
   - 2.1 Prod Sporty
   - 2.2 Prod Encore
   - 2.3 UAT Sporty
   - 2.4 UAT Encore
   （每個 instance 用相同欄位模板）

3. Performance & Cost Assessment
   - 3.1 CPU / Memory / IOPS / Connections
   - 3.2 Monthly Cost (Cost Explorer)
   - 3.3 Benchmark vs. Threshold

4. DAG Review
   - 4.1 DAG ↔ Instance Dependency Map
   - 4.2 Read/Write Patterns & Schedules
   - 4.3 Known Issues

5. Monitoring Review
   - 5.1 CloudWatch Alarm Coverage
   - 5.2 Alert Notification Setup
   - 5.3 Gaps & Blind Spots

6. Issues & Risks Summary
   （集中整理所有發現的問題）

7. Action Plan
   - P0 / P1 / P2 分級
   - Owner & Timeline
```


