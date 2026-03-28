# Kubernetes

tags: #kubernetes #k8s #devops #container #data-engineering
source: [[Isaac's Note]]

---

## K8s Fundamental

### What is K8s

Kubernetes（K8S）是一個可以幫助我們**管理微服務（microservices）的系統**，他可以自動化地部署及管理多台機器上的多個容器（Container）。

更進一步地說，Kubernetes 想解決的問題是：「手動部署多個容器到多台機器上並監測管理這些容器的狀態非常麻煩。」而 Kubernetes 要提供的解法：提供一個平台以較高層次的抽象化去自動化操作與管理容器們。

**單體架構（Monolithic Architecture）**

應用系統發展過程中，程式碼會愈來愈龐大，傳統單體架構的服務會造成很多不便：
1. 龐大且複雜的程式碼
2. 除錯、新增功能與測試都包在一起十分複雜，以低效率的開發方式進行
3. 新語言和新框架執行困難，搬移或更動耗費巨大成本

**微服務架構（Microservices Architecture）**

1. 每一個皆有商業邏輯的服務獨立出來，就不用再把資料寫至同一個資料庫而造成資料庫龐大
2. 將每個獨立的服務用最適合的語言及資料庫開發
3. 每個商業邏輯的服務為單一 VM 或單一容器
4. 獨立的微服務，例如：Web UI、金流、AI-Training

延續 docker 的概念，docker 的單位是一個容器，在 K8s 的世界中則是 Pod 而不是 container。過往微服務的概念可以透過 docker-compose 實現，而採用 K8s 的最大差異點就在於與 docker compose 的不同。

---

## K8s Terms

### Pod

Pod 是 K8s 中的最小單位，其中可以看到裡面還包了資料儲存（volume）與一或多個 container。一個 pod 通常就代表一個服務，像是提供特定功能的 API-Server，但通常不會直接去啟動或控制 pod，而是透過 **deployment** 這個抽象層去做操作。

### Node

Node 代表一個節點，也許是一個 VM 或是一個實體機，會跟 Pod 可以調用到的硬體資源限制有關係，可以當作是一台電腦，每台電腦都視為一組可以利用的 CPU 和 RAM 資源。

### Kubelet

可以當作每個 Node 都會有的守門人或管理員，負責與 Master Node 做溝通與根據配置控管 Pods 的狀態。

### 叢集（Cluster）

可以當作是最大的單位，其實就是指所有 Node 的一個集合，也就是一大堆電腦所組成的一個群體。

---

## Why K8s

K8s 與 docker compose 相比，還可以做到以下功能：

1. **根據負載（loading）去做自動化的伸縮服務數量**：可以透過提前配置好如果 CPU Loading > 80% 的時候自動增加服務（Replica）
2. **在不停機的情況下將應用程式更新進版**：Kubernetes 可以做到在不停機的條件下把容器更新，這點是 docker-compose 做不到的，如果用 docker-compose 得重啟全部容器才行
3. **多個節點上同時部署**：可以在多台主機或是多台 VM 中去運行一樣的應用服務，並且可以存取同個資料來源

---

## Setup K8s with GKE (Google Kubernetes Engine)

### Step 1. Create a GKE project

- Create a GKE project with GCP UI
- Using [Cloud Shell](https://shell.cloud.google.com/) to list all the project on GCP
- 點擊倒三角形 → 點選專案的 `PROJECT_ID` 進入專案 Terminal
- 使用 `gcloud config set compute/zone <zone>` 設定計算資源預設地區

```shell
gcloud config set compute/zone asia-east2-a
```

### Step 2. Setup K8s

使用 `gcloud container clusters create <name>` 建立 K8s cluster：

```shell
gcloud container clusters create airflow
```

取得叢集驗證（接著用 `gcloud` 將 credentials 移動到 Cloud Shell 裡來取得叢集驗證，我們就能用 `kubectl` 命令行工具對 k8s 叢集進行操作）：

```shell
gcloud container clusters get-credentials airflow
```

查看所有運行節點：

```shell
kubectl get nodes
```

### Step 3. Create Pod

要在 Kubernetes 建立元件，可以在一個 yaml 文件中描述。

建立 k8s-test 資料夾：

```shell
cd && mkdir k8s-test
```

建立 yaml 檔案：

```shell
cd ~/k8s-test && touch mypod.yaml
```

`mypod.yaml` 內容：

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: mypod
  labels:
     app: myapp
spec:
  containers:
    - name: mycontainer
      image: <your image name>
      ports:
         - containerPort: 8080
```

yaml 各欄位說明：
- `apiVersion`：該元件的版本號，根據要建立的元件而定
- `kind`：要建立的元件
- `metadata`：
  - `name`：指定該 pod 的名稱
  - `labels`：給定一個 key/value，根據標籤將 Pod 分群管理
- `spec`：負責定義 Container 詳細資訊
  - `containers`：
    - `name`：Container 的名稱
    - `image`：Container 使用的 Image
    - `ports`：指定哪些 Port 允許外部資源存取

建好了 yaml 檔案，就可以使用 `kubectl apply -f <file>` 指令，Kubernetes 會根據 yaml 檔的內容建立相應元件：

```shell
kubectl apply -f mypod.yaml
```

查看所有的 Pods：

```shell
kubectl get pods
```

查看特定 Pods：

```shell
kubectl describe pod mypod
```
