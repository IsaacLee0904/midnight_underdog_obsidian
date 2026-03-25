![[Screenshot 2026-03-24 at 5.44.07 PM.png]]

**Step1. 實際進去看 DAG 是為什麼錯誤**

**Step2. 看 `catchup` 設定**
* 如果是 catchup = False ，代表這張表下一次更新的時候是會後蓋前的，只要後面的 run 有成功，就可以直接把失敗的任務改成 Success

* 如果是 catchup = True，代表這張表每一次 run 的資料都是重要的，所以要 trace 為什麼錯誤