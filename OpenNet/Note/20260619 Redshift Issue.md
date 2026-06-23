---

---

# bi_de
--- 
**Marcus Lira [EU-BI-DE-L]** — 週五 下午 5:53

團隊，
BI 的 jobs 從 UTC 3:30 就一直持續失敗，但卻沒有人發出任何警示。
On-Call 的 @Scott Wu [TW-BI-DE] 也沒有被 tag 到。
現在又不是週末。
到底發生了什麼事？

---
**Dominykas Zenkevicius [EU-BI-DE]** — 週五 下午 6:02

我大約 UTC 7 點開始上班，Sam 在 UTC 8 點左右通知我 DAG 失敗的問題。我看到在 `bi_warehouse_alert` 頻道有幾個 DA DAG 排程 slot 的告警，但我認為未來應該要針對 data-analysis cluster 另外設一個 CPU 使用率過高的獨立告警。這在上週發生同樣問題時的討論串裡有提到過。

---
**Marcus Lira [EU-BI-DE-L]** — 週五 下午 6:02

等這次問題解決後，我們來做一份事故報告（Incident Report）。

我們需要改善這個流程。發生系統中斷時應該要全員投入，至少要先通報到正確的人。

（tag 了 @Yash Agrawal [INT-BI-DE] 和 @Trang Nguyen [INT-BI-DE]）

---
**Sam Hou [TW-BI-D]** — 週五 下午 6:06

這兩個禮拜 Redshift 的問題已經發生很多很多次了。我們需要一個解決方案。

---
**Yash Agrawal [INT-BI-DE]** — 週五 下午 6:08

我今天很早就開始工作了，從大約凌晨 1 點 UTC 就上線，但老實說我沒有看到任何明顯異常，也沒有看到任何 CPU 使用率滿載的相關告警。我會加強對這些告警的識別能力。

---
**Yash Agrawal [INT-BI-DE]** — 週五 下午 6:14

或許我們可以建立一個獨立的工具，專門即時捕捉 Redshift 發生問題時的狀況。

---
**Marcus Lira [EU-BI-DE-L]** — 週五 下午 6:32

@Sam Hou [TW-BI-D]，這就是那個一直出問題的新 data-analysis cluster。我們已經針對這個問題向 AWS 開了一張 ticket。

---
**Kevin Wei [TW-BI-DE] OOO 6/19-6/23** — 週五 晚上 11:41

分享一些關於這個問題的調查結果。

Leader Node CPU 飆升的問題大約在 UTC 02:00 再度發生（見截圖 ss1 & ss2）。

我查了上週以來這個問題發生的日期，共發生了 4 次，包含今天，全部都在 UTC 02:00 前後（ss3）。

我檢查了在 02:00 前開始跑的每日 DAG，找到一個排程在 01:30 的 DAG：`src_casino_user_segmentation`，當問題發生時它明顯跑得比較久（ss4）。

當 DAG `src_casino_user_segmentation` 裡的 task `export_prod_data` 跑得快、在 02:00 前後 30 分鐘內就完成時，CPU 沒有飆升；但如果它跑得比較久、在 02:00 之後還在跑，data-analysis cluster 的 leader node 就會 CPU 飆升。我認為我們需要進一步調查這個 DAG。
![[Pasted image 20260623092643.png]]
![[Pasted image 20260623092651.png]]
![[Pasted image 20260623092657.png]]
![[Pasted image 20260623092703.png]]

---
**Marcus Lira [EU-BI-DE-L]** — 週六 凌晨 2:03

我們需要確認每次執行時資料量是否有差異，或是其他原因。

但看這個 DAG 時，有兩件事引起了我的注意：

1. **UNLOAD 時 PARALLEL 設為 OFF**
    - 這會強制所有要匯出的資料都通過 leader node
2. **多次 delete → insert 操作**
    - 如果兩次執行之間沒有執行 VACUUM，可能會導致效能下降

---
**Scott Wu [TW-BI-DE]** — 週六 上午 10:01

Hi @Sam Hou @Marcus Lira，我們應該開始根據 dashboard 的使用情況來審視各 DAG 及其用途。這裡是一份初步報告，呈現從 dashboard/OKR 訊息追溯到上游 DA DAG 的 lineage（資料血緣）。

`DAG downstream usage` 這個 tab 中，紅色的 DAG 代表：

- 自 5 月 1 日起，dashboard 平均頁面瀏覽數 < 1
- 每日平均 DAG 執行時間 ≥ 30 分鐘

如果我們把每日頁面瀏覽數 < 1 的 DAG 執行時間全部加總，共有 84 小時，佔了 493 個 DA DAG 總執行時間的 36%。而且這份報告只涵蓋 dashboard/OKR 的角度，尚未包含那些沒有下游的 pipeline。

補充說明，有些情況會低估 DAG 的下游使用量：

1. 瀏覽主要來自 Metabase 的 cards/questions：OpenMetadata 只追蹤 dashboards，因此這些 DAG 的使用量被低估了，應該將它們遷移到 dashboards。例如 dag `src_rej_realsports_bet_breakdown_v1_p123`。
2. 沒有使用通用函式（general functions），所以無法捕捉到 lineage。

備註：DAG 執行時間不能 100% 反映 Redshift，也可能有 RDS 的貢獻。可以查看 `query submit count` 這個 tab。

---
**Scott Wu [TW-BI-DE]** — 週六 上午 10:17

我認為除非我們精簡並優化一些 pipeline，否則無法避免這個問題。主要的工作可能是需要跟利害關係人（stakeholders）溝通，讓他們了解現況。

---
**Sam Hou [TW-BI-D]** — 昨天 上午 10:45

我有幾個問題：

1. 誰在負責調查 Kevin 提到的 `src_casino_user_segmentation` DAG？但我不確定這是根本原因，還是只是 cluster 問題的影響之一。
2. 我們能否自動識別有問題的 DAG？我們現在有幾百個 DAG，需要有更好的解決方案。也許可以基於 Scott 的試算表，做出一些快速查閱的指標。
3. 哪些 DAG 沒有採用 general functions？我們能掃描出來嗎？應該讓 DA 們遵守這個規範。
4. 現在的優先順序是什麼？是 Scott 的 Lineage 分析中標紅的那些 DAG 嗎？@Scott Wu @Kevin Wei @Marcus Lira
---
**Isaac Lee [TW-BI-DE]** — 昨天 下午 2:08

Hi @Sam Hou，更新一下。現在改為透過 Sporty 和 Encore Prod 的 Airflow API 來抓取所有 active DAG，結果如下：
![[Pasted image 20260623095425.png]]
- [non-standard DAG list](https://docs.google.com/spreadsheets/d/18xM8ycFEtp4yG8J4DovGUIq6QpDZqES49gHiTJ258mE/edit?gid=1191012198#gid=1191012198)
---
**Sam Hou [TW-BI-D]** — 昨天 下午 2:16

有些是很舊的 DAG，有些是在把資料拉進 dataframe。我認為應該修正它們，但關於優先順序，不確定這是不是最緊急的，這部分交給 Marcus 來安排

---
**Marcus Lira [EU-BI-DE-L]** — 昨天 下午 5:17

@Sam Hou，我會寫一份附有行動項目的事故報告（incident report）。針對這次具體問題，調查 `src_casino_user_segmentation` 是第一優先，因為 cluster 一直在重複遇到同樣的問題。

`data-analysis` cluster 這個情況很不尋常，因為同樣的 DAG 在 `bi-warehouse` cluster 和 serverless cluster 都能正常運作。

至於 Scott 分享的那份試算表，我們會把它列入 Q3 的較長期專案來處理

---
**Marcus Lira [EU-BI-DE-L]** — 昨天 下午 6:02

我注意到在 warehouse_engineer 這邊，`parallel` 預設是 `TRUE`；但在 data-analysis repo 這邊，`parallel` 預設是 `FALSE`。

不確定這樣設計是否有特別原因，但預設不應該是 false。
![[Pasted image 20260623095959.png]]
![[Pasted image 20260623100006.png]]

---
**Marcus Lira [EU-BI-DE-L]** — 昨天 晚上 8:37

我會開始把這些 DAG 的 `parallel_on` 改成 `True`。

目前我沒看到任何可能造成影響的地方。

cc @Kevin Wei [TW-BI-DE]

---
**Dominykas Zenkevicius [EU-BI-DE]** — 昨天 晚上 10:23

等這些都完成後，我們確實可以把預設值改成 True。

---
**Marcus Lira [EU-BI-DE-L]** — 昨天 晚上 10:23

我剛 merge 了
另外，也把 `high_importance` 移回 `data-analysis` 監控了。

但現在 `bi-warehouse` 的壓力很大。

---
**Dominykas Zenkevicius [EU-BI-DE]** — 昨天 晚上 10:36

是的，還有很多 task 在排隊，看起來短時間查詢因為中等時長查詢增加而被延誤了：
![[Pasted image 20260623100124.png]]

---
**Marcus Lira [EU-BI-DE-L]** — 昨天 晚上 10:43

我已經移除了 DA (high_importance) 並且對兩邊都啟用了 CS（Concurrency Scaling），應該會恢復到原本的狀態了。

---
**Dominykas Zenkevicius [EU-BI-DE]** — 昨天 晚上 10:46

你不覺得現在 Airflow 可能是瓶頸嗎？有超過 400 個 task 在排隊等待執行——如果我們有更多 slot，它們應該可以更快被執行。

---
**Dominykas Zenkevicius [EU-BI-DE]** — 昨天 晚上 10:48

比幾個小時前好一些了，那時候大概有 500 個排程 slot，但現在還是很多。是 warehouse 的 Airflow。

---
**Marcus Lira [EU-BI-DE-L]** — 昨天 晚上 10:48

我認為是 Redshift 在拖慢它們。你可以查一下哪些 DAG 在等待嗎？感覺有點不對勁。
有沒有什麼 backfill 在追進度？感覺同時跑太多查詢了。

---
**Dominykas Zenkevicius [EU-BI-DE]** — 昨天 晚上 10:50

bi-warehouse cluster 的 CPU 使用率大約 60%，Redshift 應該還有足夠的資源來執行更多查詢。我來查一下哪些 DAG 被排程了。

---
**Dominykas Zenkevicius [EU-BI-DE]** — 昨天 晚上 10:55

以下是一些有 task 排程等待執行的 DAG 清單：

- `afbet_facts.t_facts_event_info` — 比排程晚了超過 3 小時，正在追進度，task 卡在 scheduled 狀態
- `afbet_instant_win.t_instant_win_bet` — 同上
- `afbet_instant_win.t_instant_win_bet_detail` — 同上
- `afbet_instant_win.t_instant_win_bet_detail_new` — 同上
- `afbet_instant_win.t_instant_win_ticket_v2` — 同上
- `afbet_instant_win.t_instant_win_user_bet_builder_selection_pt` — 同上

看起來現在瓶頸是 Airflow default pool 的 slot 數量不足。

---
**Marcus Lira [EU-BI-DE-L]** — 昨天 晚上 10:56

試著多開一些 slot，但我覺得最後還是會排到隊伍裡。

---
 **Dominykas Zenkevicius [EU-BI-DE]** — 昨天 晚上 10:57

我可以再加 50 個 slot 看看有沒有幫助。

---
**Marcus Lira [EU-BI-DE-L]** — 昨天 晚上 10:57

我們也可以試試重啟 cluster，感覺有什麼東西卡住了。

---
**Dominykas Zenkevicius [EU-BI-DE]** — 昨天 晚上 10:59

增加 slot 數量後，排程中的 task 數量開始穩定下降，我會持續監控 Redshift CPU 使用率。

---
**Dominykas Zenkevicius [EU-BI-DE]** — 昨天 晚上 11:04

雖然我把 slot 數量增加到 150，但 Airflow 似乎有個硬上限是 120（可能是在 Airflow config 某處設定的）。嘗試繼續增加到 175，但還是無法超過 120 個 slot。

---
**Marcus Lira [EU-BI-DE-L]** — 昨天 晚上 11:05

有一個 ANALYZE 在跑：

```
ANALYZE afbet_patron_ng.logs_patron_fe_behav;
```

---
**Dominykas Zenkevicius [EU-BI-DE]** — 昨天 晚上 11:18

很多 Airflow task 還是卡在 scheduled 狀態，running slot 數量無法超過 125-127。

---
**Marcus Lira [EU-BI-DE-L]** — 昨天 晚上 11:23

我已經把那個 ANALYZE 給 kill 掉了，看看情況。如果沒有改善，我們就重啟。

---
**Dominykas Zenkevicius [EU-BI-DE]** — 昨天 晚上 11:25

那些 task 在 Airflow 裡卡在 scheduled 狀態，根本還沒有跑到 Redshift 上。增加 slot 上限應該能解決這個問題，但我不知道要怎麼在 AWS EKS 上存取 Airflow config 來增加 running slot 的上限。

---
**Marcus Lira [EU-BI-DE-L]** — 昨天 晚上 11:26

這是因為 Redshift 滿了，查詢跑太久了……在等待中。

---
**Marcus Lira [EU-BI-DE-L]** — 昨天 晚上 11:33

看起來排隊數量在下降了。

---
**Dominykas Zenkevicius [EU-BI-DE]** — 昨天 晚上 11:37

排程中的 task 數量首次降到 340 以下了！🎉

---
**Marcus Lira [EU-BI-DE-L]** — 昨天 晚上 11:48

看起來現在在跑更多查詢了，我去吃點東西，等等回來看看。

---
**Dominykas Zenkevicius [EU-BI-DE]** — 昨天 晚上 11:55

排程中的 task 數量繼續下降，現在大約 250 了。

---
**Marcus Lira [EU-BI-DE-L]** — 昨天 晚上 11:55

這是在我 kill 掉 analyze job 和一些 idle session 之後發生的。

---
**Kevin Wei [TW-BI-DE] OOO 6/19-6/23** — 昨天 晚上 11:58

Hi @Marcus Lira @Dominykas Zenkevicius，我們每天還有 VACUUM DELETE job 也會消耗資源，我剛把它標記為成功（跳過），這樣應該可以給 bi-warehouse 更多空間來追進度。

---
**Marcus Lira [EU-BI-DE-L]** — 今天 凌晨 12:00

呼！看起來恢復正常了！😅

---
**Kevin Wei [TW-BI-DE] OOO 6/19-6/23** — 今天 凌晨 12:01

另外，`app_report_airflow_rw` 這個 user 是什麼時候有的？它的 role 是什麼？有一個查詢跑了 12 小時……

看起來我們需要針對所有 role 設定長時間查詢自動中止（long query abort）的監控規則。
![[Pasted image 20260623100741.png]]

--- 
**Marcus Lira [EU-BI-DE-L]** — 今天 凌晨 12:02

那是我剛剛 kill 掉的其中一個 idle session。

---
**Marcus Lira [EU-BI-DE-L]** — 今天 凌晨 12:02

我也 kill 了一個來自 `da_trading` 的，還有其他來自 `app_airflow_rw` 的。

---
**Kevin Wei [TW-BI-DE] OOO 6/19-6/23** — 今天 凌晨 12:07

根據我的經驗，通常導致 `bi-warehouse` cluster 變慢、warehouse DAG 執行時間拉長甚至延遲的原因有：

1. 新的 DA job 從很久以前開始持續做 backfill
2. 新的 DA job 高併發，同時跑很多查詢
3. 個人帳號或團隊帳號（如 `da_trading`）跑臨時的長時間查詢（adhoc long query）

---
