tags : #cicd #gitlab #devops #data-engineering
source : Notion

---

## Introduction of CICD

### Why we need CICD?

在隨著服務、網路、容器化使用越來越發達也越來越複雜的情況，系統架構的集成與部屬成為了新的挑戰，從而使開發者通過更先進的 Devops 流程來優化部屬流程，而 GitLab CI/CD 就是在這樣的脈絡下產生。

### What is CICD?

講的比較簡單一點就是將上程式的流程自動化，自動 build code、執行 unit test、自動更新線上服務...所有反覆步驟都轉為自動化執行，實際上 `CI` 與 `CD` 應該要拆成兩個部份來看。

### 持續整合 / CI (Continuous Integration)

流程：
- **程式建置 (build)**：開發人員在每一次的 Commit & Push 後，都能夠於統一的環境自動 Build 程式，透過此一步驟可以避免每個開發人員因本機的環境＆套件版本不相同，造成 Service 異常
- **程式測試 (test)**：當程式編譯完成後，將會透過 unit test 測試新寫的功能是否正確，或者確認是否有影響到現有功能

目的：
- 降低人為疏失風險
- 減少人工手動的反覆步驟
- 進行版控管制
- 增加系統一致性與透明化
- 減少團隊 loading

### 持續佈署 / CD (Continuous Deployment)

流程：
- **部署服務 (deploy)**：透過自動化方式，將寫好的程式碼更新到機器上並公開對外服務，另外需要確保套件版本＆資料庫資料完整性，也會透過監控系統進行服務存活檢查，若服務異常時會即時發送通知告至開發人員

目的：
- 保持每次更新程式都可順暢完成
- 確保服務存活

---

## GitLab Workflow Introduction

### GitLab CICD Fundamental

GitLab 的 CICD 由 **GitLab Runner** 與 **.gitlab-ci.yml** 兩個部分組成，前者為 CICD pipeline 的運行環境，後者則是一個 YAML 檔案，以結構化的方式宣告一條 pipeline，其中定義了 stages, job 等，並且基於 pipeline 依附在 branch 之上，換句話說需要**透過 merge 到對應的 branch 來觸發 CI/CD**。

**GitLab Runner**

Step1. 安裝 runner（Docker）：

```bash
docker run -d --name gitlab-runner --restart always \
    -v /src/gitlab-runner/config:/etc/gitlab-runner \
    -v /var/run/docker.sock:/var/run/docker.sock \
    gitlab/gitlab-runner:v14.1.0
```

Step1. 安裝 runner（Linux）：

```bash
curl -L https://packages.gitlab.com/install/repositories/runner/gitlab-runner/script.deb.sh | sudo bash
sudo apt-get install gitlab-runner
```

Step2. 註冊 runner：

```bash
sudo gitlab-runner register
```

### GitLab Workflow 10 Steps

GitLab 公司認為從創意發想到產品上線開始收集意見為止，我們可以把軟體開發的流程大致分為 10 個步驟：

1. **Idea**：除了面對面的實體會議，線上的聊天討論空間中的某個想法，也許亦有機會成為產品創意發想的養分
2. **Issue**：有了天馬行空的 Idea 之後，接著要將其轉變成更具體明確的 Story 或 Issue。GitLab 提供了完整的 Issue Tracking 功能
3. **Plan**：延續步驟二，當 Story 或 Issue 被逐一建立完畢之後，在進入實際的程式開發工作之前，要先決定工作的優先順序。GitLab 的 Issue Tracker 可以直接搖身一變成為 Issue Board（任務版）
4. **Code**：有了計畫之後，下一步當然就是動手撰寫程式
5. **Commit**：程式撰寫完畢之後，即可送入版本控制之中
6. **Test**：當程式碼被提交至版本控制系統之後，接著就是測試。這也就是 GitLab CI 服務上場的地方了
7. **Review**：當 CI 自動化建置、測試都通過之後，在將程式碼 Merge 至主要的 Branch 之前，需要先進行 Review 的動作
8. **Staging**：程式順利 Merge 之後，即可將程式自動部署至 Staging 或 Production-like 的環境。GitLab CI 擁有名為 **Auto DevOps** 的功能，讓 CI/CD 自動化部署更加自動、順暢與便利
9. **Production**：當程式在 Staging 環境亦順利通過驗證，最後即可部署至 Production 環境供使用者使用
10. **Feedback**：這是最後、也是非常重要的一個步驟，它能幫助產品及團隊自身得以繼續不斷的「持續改善」

### GitLab Flow

GitLab Flow 是 GitLab 公司由 GitHub Flow 再發展而來的，保留了 GitHub Flow 的分支策略，依然有 feature branch 與 master branch，但在 master 之外再增加專門用來配合交付與部署的 branch：

- **Production branch**：用來幫助團隊明確知道目前正式上線的程式碼是哪一個版本。開發進度依然按照 Github Flow 的做法，由 feature branch 合併至 master branch，當程式要正式上線時，才根據要交付的範圍，從 master branch 合併至 production branch，同時在 production branch 即可搭配 GitLab CI 功能觸發 CI Job - production deploy
- **Environment branch**：可以根據不同的部署環境額外建立對應的 branch，例如增加一個 pre-production branch 對應 pre-production 環境
- **Release branch**：如果你的程式並非是直接部署於某個環境，而是直接對外發佈（release），那麼 GitLab Flow 的作法是根據每次的 release 建立對應的 release branch

---

## GitLab Auto DevOps

### Introduction of GitLab Auto DevOps

Auto DevOps 的背後是仰賴 GitLab 官方事先預備好的眾多 **GitLab CI Templates**，這些 Templates 能夠根據 Project 內容，自動產生多種情境的 CI/CD Pipeline。

`Auto-DevOps.gitlab-ci.yml` 是 GitLab Auto DevOps 自動產生 CI/CD Pipeline 的起點，其中定義了 Pipeline 的 Stages 以及要引用哪些 CI Template。

首先是定義 `image:` 與 `variables:`：

```yaml
## 定義 image
image: alpine:latest

## 定義 variables
variables:
  CS_DEFAULT_BRANCH_IMAGE: $CI_REGISTRY_IMAGE/$CI_DEFAULT_BRANCH:$CI_COMMIT_SHA

  POSTGRES_USER: user
  POSTGRES_PASSWORD: testing-password
  POSTGRES_DB: $CI_ENVIRONMENT_SLUG

  DOCKER_DRIVER: overlay2
  ROLLOUT_RESOURCE_TYPE: deployment
  DOCKER_TLS_CERTDIR: ""
```

接著是 `stages:`，官方預先規劃了共 14 個 Stage：

```yaml
stages:
  - build
  - test
  - deploy
  - review
  - dast
  - staging
  - canary
  - production
  - incremental rollout 10%
  - incremental rollout 25%
  - incremental rollout 50%
  - incremental rollout 100%
  - performance
  - cleanup
```

> 在 GitLab CI 中，CI Job 一定要分配 `stage:`，但 `stages:` 裡面不一定要有 CI Job，因此在 Pipeline 初期規劃階段，你其實可以大膽的列出所有想要的 `stages:`，然後隨著 Pipeline 的逐步改善，將每個 Stage 的 CI Job 慢慢補上。

`workflow:` 類似程式語言中的 `if / else`，在 GitLab CI 我們可以利用 `workflow:` 設置不同的條件，藉此控制 GitLab CI 判斷哪些情境之下，才需要產生 CI/CD Pipeline：

```yaml
workflow:
  rules:
    - exists:
        - Dockerfile
    - exists:
        - package.json
    - exists:
        - composer.json
        - index.php
    - exists:
        - Gemfile
    - exists:
        - requirements.txt
        - setup.py
        - Pipfile
```

最後是利用 `include:` 引用了多個 Template：

```yaml
include:
  - template: Jobs/Build.gitlab-ci.yml
  - template: Jobs/Test.gitlab-ci.yml
  - template: Jobs/Code-Quality.gitlab-ci.yml
  - template: Jobs/Code-Intelligence.gitlab-ci.yml
  - template: Jobs/Deploy.gitlab-ci.yml
  - template: Security/DAST.gitlab-ci.yml
  - template: Jobs/Container-Scanning.gitlab-ci.yml
  - template: Jobs/Dependency-Scanning.gitlab-ci.yml
  - template: Jobs/SAST.gitlab-ci.yml
  - template: Jobs/Secret-Detection.gitlab-ci.yml
```

### More about Include Section

透過 `include:` 讓我們可以實現依據不同的層級，定義不同的 `.yml`：

```yaml
# 指的是使用 gitlab server 上的 template
include:
  - template: Jobs/Build.gitlab-ci.yml

# 使用 local 表示引用的是存放在同一個 Project 的 .yml
include:
  - local: '/Jobs/build.yml'
```

實務上我們可以將這些 CI Job 用另一個 Project 管理，改用 `include:project` 的方式引用：

```yaml
include:
  - project: 'my-pipeline-template'
    ref: v2021.1
    file:
      - '/Jobs/build.yml'
      - '/Jobs/unit-test.yml'
```

---

## Stages & Keywords

### Stages

`stages` 是一個全局的關鍵字，用於定義一個 pipeline 包含哪些階段，定義於 `.gitlab-ci.yml` 頂部，並且 job 的執行順序是根據 stages 的定義順序。

> 如果沒有定義，則帶入 default stages：`.pre` > `build` > `test` > `deploy` > `.post`

在 `jobs` 中則用 `stage` 來宣告每個 `jobs` 的階段，用 `script` 定義該 `jobs` 需要執行的 script：

```yaml
validate_migrations:
  <<: *goose_base
  stage: validate
  script:
    - echo "Validating migration files..."
    - pwd
    - ls -la migrations/
    - goose -dir ./migrations validate
    - echo "Checking for dangerous operations..."
    - grep -r "DROP\|TRUNCATE\|DELETE" ./migrations/ || echo "No dangerous operations found"
  rules:
    - if: $CI_COMMIT_BRANCH
```

### Keyword in .gitlab-ci.yml

1. **Cache**：`cache` 可以用來管理 pipeline 中的緩存、上傳和下載，可以達到將不同 `jobs` 中共用的文件或資料夾緩存起來，讓後續的任務來使用，需注意的是緩存的文件路徑必須是根目錄的相對路徑
2. **Tags**：`tags` 用於指定任務使用哪一個 runner 來執行，透過在建立 runner 的時候填寫一個或數個 tags，runner 應該盡量保持一對一關係
3. **Variables**：在 pipeline 中，可以用 `variables` 定義一些參數，這些參數會默認為環境變數，變數的官方寫法是大寫英文字母加上下底線，例如 `UAT_CLICKHOUSE_HOST`

### More about Workflow Section

`workflow:` 可以使用的 `rules:` 共有下面三種：

- **`rules:if`**：可以做簡單的 if 判斷，一般來說會搭配 Variables 做條件判斷，例如 `if: '$SKIP_NPM == true'`
- **`rules:changes`**：根據該次 Commit 中，特定檔案是否有被異動，判斷是否要產生 Pipeline
- **`rules:exists`**：根據 Project 內是否含有特定檔案，判斷是否要產生 Pipeline

```yaml
workflow:
  rules:
    # 當 Project 的根路徑之下有 Dockerfile 時，就要產生 Pipeline
    - exists:
        - Dockerfile
    # 當 Project 的根路徑之下有 package.json 時，就要產生 Pipeline
    - exists:
        - package.json
    # 當 Project 的根路徑之下有 composer.json 或 index.php 兩者其一時，就要產生 Pipeline
    - exists:
        - composer.json
        - index.php
    # 當 Project 的根路徑之下有 Gemfile 時，就要產生 Pipeline
    - exists:
        - Gemfile
```
