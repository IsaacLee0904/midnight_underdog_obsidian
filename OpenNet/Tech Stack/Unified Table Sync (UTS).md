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

---

## 7. Deployment Notes (K8s)

UTS 以兩個 runtime instance 部署（相同 codebase）：

### 7.1 Sporty Deployment

- 僅處理 DA modal 提交
- 轉發至 OpenNet FastAPI
- **不**發佈至 `bi_de`

### 7.2 OpenNet Deployment

- 部署 FastAPI endpoint
- 發佈至 DE 審核頻道
- 處理 DE 操作
- 處理 callback
- 建立 PR

**Configuration**

K8s 使用：
- `ConfigMap`：非敏感環境變數
- `Secret`：token 與 secret

---

## 8. How to Modify Safely (Maintenance Guide)

### 8.1 Slack Timeout Rule

Slack 要求在約 3 秒內完成 `ack()`。

> ✅ 所有耗時操作必須移至 background thread 或非同步 task queue（未來升級方向）

### 8.2 When Editing the DA Modal

在 `app.py` 中更新：
- View blocks
- 欄位解析邏輯
- View 中顯示的驗證錯誤

確認事項：
- 只有當 `backfill=yes` 時，backfill start date 才為必填
- Schedule cron 可為選填

### 8.3 When Editing DE Workflow

在 `de_review.py` 中更新：
- Slack blocks 格式
- Approve / edit / reject handler
- 允許的 DE 使用者清單邏輯

重要事項：
- 永遠立即執行 `ack()`
- 對未授權使用者使用 ephemeral message

### 8.4 When Editing GitHub Behavior

在 `github_uts.py` 中更新：
- repo 路徑
- branch 命名規則
- PR title/body template
- 「若已 managed 則跳過」邏輯

### 8.5 When Editing DAG Generation Logic

在以下檔案中更新：
- `uts_callback_handler.py`
- `dag_template.py.tmpl`
- `dag_template_backfill.py.tmpl`

注意事項：
- schedule 字串的正確性
- `dag_id` 命名規範
- backfill 的 cursor 更新邏輯

---

## 9. Debugging Guide (Common Issues)

### 9.1 "DE approves but Slack shows error"

**可能原因：** Slack Interactivity URL 指向錯誤的 runtime（DevOps ngrok vs local）

**解決方式：**
- 確認 Slack app 設定指向正確的 endpoint
- 同一個 Slack app configuration 只能有一個 active endpoint

### 9.2 Airflow run succeeds instantly (0 sec duration)

**常見原因：**
- `logical_date` < `start_date`
- 因 start date 限制導致 task 未被排程

**解決方式：**
- 將 `start_date` 設為過去的時間，或使用 `logical_date ≥ start_date` 觸發

### 9.3 Backfill cursor not advancing

**檢查 Redshift：**
- `uts_backfill_state` 中是否有對應資料列
- `cursor_start_ts` 是否 < 今天
- cursor 更新只在最後一個 country 完成後才觸發
