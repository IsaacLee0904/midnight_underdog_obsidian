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
- DA 的入口（modal / slash command）
```bash
/table_sync
```
- 將 request payload 透過 FastAPI 轉發至 OpenNet runtime

<mark style="background: #ADCCFFA6;">OpenNet Slack App</mark>
- 將申請發佈至 DE 審核頻道（`bi_de`）![[Screenshot 2026-03-30 at 11.25.22 AM.png]]
- 處理 DE 操作（Approve / Edit & Approve / Reject）
- 觸發 Airflow
- 處理 Airflow callback
- 為 DAG 建立 GitHub PR

**FastAPI callback server（部署於 OpenNet runtime）**
* Receives
	- Sporty app 轉發的 DA request
	- Airflow callback payload（`ddl_result`）

非同步分派工作

**Airflow**
- 執行 `uts_ddl_bootstrap` DAG（bootstrap）
- schema/table DAG 架構就緒後，回呼 UTS server

**GitHub**
- 接收 callback handler 建立的 PR

**Meta DB**
- 用於 metadata 查詢與 config 展開

---

## 3. End-to-End Flow (What Happens In Production)

### 3.1 DA Submission Flow (Sporty → OpenNet)

![[UTS Sequence 1.png]]

**步驟說明**

1. DA 觸發流程（slash command / modal）
2. Modal 開啟
3. DA 提交表單
4. Slack 要求在約 3 秒內 `ack()` → app 立即回應 success view
5. Sporty App 將 payload 轉發至 OpenNet FastAPI：`POST /internal/uts/da_submit`
6. FastAPI 快速回傳 200
7. OpenNet app 非同步將申請發佈至 DE 審核頻道（`bi_de`）
8. 含操作按鈕的 DE thread 建立完成

### 3.2 DE Review & Automation Flow (Approve → Airflow → Callback → PRs)

![[UTS Sequence 2.png]]

**步驟說明**

1. DE 點擊 Approve / Edit & Approve / Reject
2. OpenNet app 接收 action payload
3. App 立即執行 `ack()`
4. App 針對所選的 brand/env 觸發 Airflow bootstrap DAG
5. Airflow 執行完畢並發送 callback：`POST /internal/uts/ddl_result`
6. FastAPI 回傳 200
7. Callback handler 非同步處理結果
8. Callback handler 建立 GitHub PR
9. PR 連結回傳至同一個 Slack thread

---

## 4. Repository Structure & File Responsibilities

> ✅ 所有變更都應確保 Slack 的即時回應性：永遠先執行 `ack()`，再將耗時操作移至非同步處理。

### 4.1 `app.py` — Slack App Entrypoint (Sporty + OpenNet)

**職責**
- 主要 Bolt app 入口
- 處理 Slack slash command、modal 開啟、modal 提交
- 在 OpenNet runtime 中啟動 FastAPI server

**核心功能**
- 驗證與建構 `uts_da_request` payload
- Slack UI 更新（success/error view）
- 轉發邏輯：
  - Sporty app 將 DA submit 轉發至 OpenNet FastAPI
  - OpenNet 直接發佈至 DE 審核頻道
- 啟動 server 並掛載以下 route：
  - `/health`
  - `/internal/uts/da_submit`
  - `/internal/uts/ddl_result`

### 4.2 `de_review.py` — DE Review Workflow Logic

**職責**
- 建構格式化的 DE review 訊息與 Slack blocks
- 發佈至 `bi_de`
- 處理 DE 操作：Approve、Edit & Approve、Reject、Proceed / Abort

**核心功能**
- DE 專屬操作的授權驗證
- `ack()` 後非同步呼叫 Airflow REST API，針對每組（brand, env）觸發 bootstrap DAG
- 操作後更新 thread 訊息

**重要行為**
- `ack()` 之前不進行耗時操作
- Airflow 觸發必須在 `ack()` 之後或非同步執行

### 4.3 `uts_callback_handler.py` — Callback Processing Orchestrator

**職責**
- 執行 Airflow callback 後的「post-bootstrap」步驟

**核心功能**
- 解析 callback payload
- 驗證必填欄位
- 使用 template 產生 DAG 內容
- 判斷哪些 DAG 需要 PR 更新、哪些可跳過
- 呼叫 GitHub PR 建立邏輯
- 將 PR 連結回傳至 Slack thread

### 4.4 `github_uts.py` — GitHub PR Creation & Repo Updates

**職責**
- 管理所有 GitHub API 互動
- 建立含有產生 DAG 內容的 PR

**核心功能**
- 建立 branch
- Commit 產生的 DAG 檔案
- 開啟 PR
- 回傳 PR URL

**特殊邏輯**
- 偵測 DAG 是否已存在且標記為 UTS-managed
- 若已為 UTS-managed，則避免建立重複 PR

### 4.5 `dag_template.py.tmpl` — Normal (Non-Backfill) DAG Template

**職責**
- 用於標準 sync DAG 產生的 template

**核心功能**
- 包含 UTS-managed marker：`UTS_MANAGED_DAG = True`
- 提供排程定義
- 依據 logical interval 執行增量同步

### 4.6 `dag_template_backfill.py.tmpl` — Backfill DAG Template

**職責**
- 用於 backfill DAG 產生的 template

**核心功能**
- 有狀態的週進度 backfill
- 使用 cursor 邏輯：
  - start time = state cursor
  - end time = start + 1 week
  - 完成一個時間窗口後更新 cursor

**目前設計說明**
- Backfill cursor 儲存於 Redshift table（非 Airflow Variables）
- 只有在最後一個 country 完成後才更新 `cursor_start_ts`

---

## 5. FastAPI Endpoints (OpenNet Runtime)

### 5.1 `/health`

確認服務是否正常運行的 healthcheck endpoint。

### 5.2 `/internal/uts/da_submit`

- **使用者：** Sporty app
- **目的：** 將 DA 申請轉發至 OpenNet 流程

**預期 payload 欄位**
- `requested_by`
- `brands`
- `envs`
- `schema` / `db`
- `tables`
- `countries`
- backfill 相關欄位
- schedule 相關欄位
- `notes`

**重要事項**
- 必須驗證 secret header（例如 `X-UTS-Secret`）
- 必須快速回傳 200
- Slack 發佈應在 background thread 中執行

### 5.3 `/internal/uts/ddl_result`

- **使用者：** Airflow callback
- **目的：** 將 bootstrap DAG 結果傳遞至 app 以建立 PR

**預期 payload 類型：** `uts_ddl_bootstrap_result`

**重要事項**
- 必須驗證 secret header
- 必須包含 Slack thread 識別資訊：
  - `slack.channel_id`
  - `slack.thread_ts`
- 工作以非同步（thread）方式執行，避免 Slack / Airflow timeout

---

## 6. UTS-Managed DAG Rules

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
