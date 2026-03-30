tags : #OpenNet #data-engineering #etl
source : OpenNet

---
##  What Problem This App Solves

DA 團隊經常需要建立新的 Redshift table sync pipeline，並可能包含 backfill 與排程設定

<mark style="background: #FFB86CA6;">UTS Slack App 的功能</mark>
* A **self-serve DA request form**
* A **DE review & approval workflow**
* Automated workflow
	* triggering of Airflow bootstrap DAGs
	* DAG generation from templates
	* creation of GitHub PRs
	* posting PR links back into Slack threads
---

## High-Level Architecture

![[Pasted image 20260330111503.png]]

> <span style="color:rgb(184, 191, 193)">UTS 跨兩個 Slack workspace 運行，因此使用兩個 Slack app</span>

**<mark style="background: #ADCCFFA6;">Sporty Slack App</mark>
- <span style="color:rgb(255, 0, 0)">bi_job</span> 的 Slack Channel
- DA 的入口，透過指令可以創建需求單
```bash
/table_sync
```
- 將 request payload 透過 FastAPI 轉發至 OpenNet runtime

<mark style="background: #ADCCFFA6;">OpenNet Slack App</mark>
- 將申請發佈到 <span style="color:rgb(255, 0, 0)">bi_de</span> 給 DE 審核![[Screenshot 2026-03-30 at 11.25.22 AM.png]]
- 觸發 Airflow
- 處理 Airflow callback
- 為 DAG 建立 GitHub PR

<mark style="background: #ADCCFFA6;">FastAPI callback server</mark>
* 部署於 OpenNet runtime
- Sporty app 轉發的 DA request
- Airflow callback payload（`ddl_result`）
> <span style="color:rgb(184, 191, 193)">是非同步分派工作</span>

<mark style="background: #ADCCFFA6;">Airflow</mark>
- 執行 `uts_ddl_bootstrap` DAG（bootstrap）
- schema / table DAG 架構就緒後，回呼 UTS server

<mark style="background: #ADCCFFA6;">GitHub</mark>
- 接收 callback handler 建立的 PR
---

## End-to-End Flow 
### Script Workflow
![[Pasted image 20260330114314.png]]

> 所有的任務都會先回應 `ack()`
> 
> * Slack 規定你必須在 **3 秒內** 回 `ack()`，否則 Slack 會認為你的 server 沒有回應，然後跳出錯誤訊息給使用者，所以先 `ack()`，讓 Slack 收到再把耗時的工作 EX. Airflow、Create PR 弄到背景做非同步處理

### DEs Workflow
![[UTS workflow|800]]

---

## UTS-Managed DAG Rules

為防止重複或衝突的 PR：

**UTS-managed marker**

產生的 DAG 包含：
```python
UTS_MANAGED_DAG = True
```

**行為規則**
- 若 DAG 已存在 **且** 含有 `UTS_MANAGED_DAG = True` → UTS 不會重新建立非 managed 版本
- 此機制防止重複，並維持所有權邊界

#### Reference
[Confluence Page](https://opennetltd.atlassian.net/wiki/spaces/DET/pages/4282351705/Unified+Table+Sync+UTS+Slack+App+Developer+Maintainer+Guide)
