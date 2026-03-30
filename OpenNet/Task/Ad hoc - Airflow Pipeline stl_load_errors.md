![[Screenshot 2026-03-30 at 4.28.48 PM.png]]

有時候會在 pipeline 上遇到 <span style="color:rgb(255, 0, 0)">stl_load_errors</span> 的錯誤訊息，代表 RDS 與 Redshift 的欄位沒有對上，這時候需要去確認並處理

Step1. 先去看 Redshift log
```sql
SELECT * 
FROM stl_load_errors 
ORDER BY starttime DESC 
LIMIT 10;
```

![[Screenshot 2026-03-30 at 4.48.40 PM.png]]

這個例子是