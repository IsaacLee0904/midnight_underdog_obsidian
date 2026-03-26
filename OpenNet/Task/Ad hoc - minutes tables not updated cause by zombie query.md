![[Screenshot 2026-03-26 at 1.08.05 PM.png]]

**Step1. 因為這些 table 都是分鐘級更新的 table，理論上不應該這麼久還沒更新**

![[Screenshot 2026-03-26 at 1.13.38 PM.png]]
可以看到跳 alart 的時候是 15:29，但這些 table 的最後更新時間已經是 14:00 左右了，而且一次出現大量的 table 失敗，代表有東西卡住了 Redshift 效能

**Step2. 進到 Redshift monitoring 看**

1. Link : [Redshift monitoring](https://eu-central-1.console.aws.amazon.com/redshiftv2/home?region=eu-central-1#/cluster-details?cluster=sporty-pub-prod-bi-warehouse&tab=queries)
2. Query monitoring 的 tab 看近一小時的時間區間

**Step3. 找出 long query 並砍掉**

![[Pasted image 20260326131830.png]]
從圖中可以看到 `src_ops_linked_accouts_d_v01` 的查詢跑了非常久的時間，加上上面還有 zombie query 的告警，可以判斷是因為這個查詢導致的
![[Screenshot 2026-03-26 at 1.20.17 PM.png]]