---
cluster: sporty-pub-prod-bi-main
alias: "bi-report-o1, bigdata-o1"
engine: Aurora MySQL
instance_type: db.r6g.12xlarge
multi_az: true
cpu_writer: 34
environment: sporty-prod
status: available
---

## Notes
- Writer CPU 34%, 18 sessions
- Reader CPU 3.83%