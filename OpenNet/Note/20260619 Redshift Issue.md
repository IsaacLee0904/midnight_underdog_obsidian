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

### **Sam Hou [TW-BI-D]** — 昨天 上午 10:45

我有幾個問題：

1. 誰在負責調查 Kevin 提到的 `src_casino_user_segmentation` DAG？但我不確定這是根本原因，還是只是 cluster 問題的影響之一。
2. 我們能否自動識別有問題的 DAG？我們現在有幾百個 DAG，需要有更好的解決方案。也許可以基於 Scott 的試算表，做出一些快速查閱的指標。
3. 哪些 DAG 沒有採用 general functions？我們能掃描出來嗎？應該讓 DA 們遵守這個規範。
4. 現在的優先順序是什麼？是 Scott 的 Lineage 分析中標紅的那些 DAG 嗎？@Scott Wu @Kevin Wei @Marcus Lira