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

這個例子是 `ticket_info` 的欄位超過了原本設定的 var 數值，但也到了 Redshift 的上限，所以透過去 pipeline 把字串截斷來處理

> [!important]
> Redshift 的 `VARCHAR(n)` 中的 `n` 是以 **bytes（位元組）** 計算，不是字元數
> 因此一個中文字（UTF-8）佔 3 bytes，emoji 佔 4 bytes
> 例如 `VARCHAR(100)` 最多只能存約 33 個中文字，而不是 100 個

Step2. 修正錯誤
```python
def unzip_and_to_json(ticket_info: str):

	import base64
	import zlib
	import gzip
	
	decode = base64.b64decode(ticket_info)
	
	if ticket_info[:1] != "H":
		inflater = zlib.decompressobj()
		bytes_ = inflater.decompress(decode)
		result = bytes_.decode("utf-8")
	
	else:
		result = gzip.decompress(decode).decode("utf-8")
	
	utf8_bytes = result.encode("utf-8")
	if len(utf8_bytes) > 65535:
		result = utf8_bytes[:65535].decode("utf-8", errors="ignore")
	
	return result
```