Due to currently Prod Sporty Redshift workload issue. DE team plan to [switch personal ad-hoc query to additional serverless](https://opennetltd.atlassian.net/jira/software/projects/BDE/boards/117?jql=assignee%20%3D%20712020%3A993acc76-02e5-4173-9e2b-61373ff1764c&selectedIssue=BDE-1425). After grant permission, most of user login with TablePlus can only see `public` schema (as screenshot), but are able to query normally. Except Alex and he is using [DataGrip](https://www.jetbrains.com/datagrip/) which can see all schemas correctly.

![[Pasted image 20260702110426.png]]

### Serverless and Datashare 

![[Screenshot 2026-07-02 at 11.13.21 AM.png]]


### Troubleshooting 

#### TablePlus conn with Redshift Driver

Step1. 測試是否可以看到 `SVV_ALL_SCHEMAS`
![[Screenshot 2026-07-02 at 11.23.08 AM.png]]

什麼都沒有顯示，但是可以查詢 `SVV_ALL_SCHEMAS` 的
```sql
-- show all the datashare
SELECT * FROM SVV_ALL_SCHEMAS;

-- show each schema and it belong database
SELECT database_name, schema_name
FROM SVV_ALL_SCHEMAS
WHERE database_name IN ('bi_warehouse', 'bi_report')
ORDER BY database_name, schema_name;
```
![[Screenshot 2026-07-02 at 11.32.23 AM.png]]

-> 代表 DA 帳號是有 `SVV_ALL_SCHEMAS` 權限的，因此也代表 Tableplus 不是透過查詢 SVV_ALL_SCHEMAS 來拿到 UI 上的顯示資訊

Step2. 透過連線 dev db 來看 TablePlus 在顯示 UI 時使用的 query
```SQL
SELECT nspname FROM pg_catalog.pg_namespace;

SELECT 
  pg_catalog.pg_get_userbyid(p.proowner) as owner,
  p.oid AS oid,
  n.nspname AS function_schema,
  p.proname AS function_name,
  CASE。
     WHEN p.proisagg THEN'aggregate'
     WHEN p.prorettype='pg_catalog.trigger'::pg_catalog.regtypeTHEN'trigger'
     ELSE'function'
  END AS function_type 
FROM pg_catalog.pg_proc p LEFT JOIN pg_catalog.pg_namespace n ON n.oid=p.pronamespace;

SELECT 
    tablename as table_name,
    schemaname as table_schema,
    'TABLE' as table_type 
FROM pg_tables 
UNION 
SELECT 
    viewname as table_name,
    schemaname as table_schema,
    'VIEW' as table_type 
FROM pg_views 
UNION 
SELECT 
   tablename as table_name,
   schemaname as table_schema,
   'TABLE' as table_type 
FROM svv_external_tables;
```