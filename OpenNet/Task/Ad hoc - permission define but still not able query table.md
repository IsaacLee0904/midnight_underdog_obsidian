有時候已經在 repo 裡面定義了某個 schema 的讀取權限，但還是遇到 table query deney 的問題
![[Pasted image 20260717113437.png]]

那是因為在 `t_db_meta` 裡面的 matching 錯了
![[Pasted image 20260717113537.png]]
有時候 country 會是 `pub` 但實際上應曬 $CC 或是剛好相反