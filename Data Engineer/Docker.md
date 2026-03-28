# Docker

tags: #docker #container #devops #data-engineering
source: [[Isaac's Note]]

---

## Docker Fundamental

### Why use Docker

- Docker 讓環境統一變得更容易，這有助於設計持續整合、持續部署的架構
- 只要雲端服務或機器支援 Docker，就能運行 Docker 包裝好的服務
- Docker 使用 `Dockerfile` 的純文字檔做為建置 image 的來源，這代表 Docker 實現了環境即程式碼（Infrastructure-as-code，IaC），環境資訊可以被版控系統記錄
- 同為虛擬化技術，Docker（或指 container）的資源使用率較 VM 好，它直接跟底層共用作業系統，不需中間再隔一層作業系統虛擬層，因此啟動 container 非常快
- Docker 應用場景橫跨開發與維運，也是 DevOps 熱門技能之一

### Docker Terms

| 英文 | 中文 |
|------|------|
| build | 建置 |
| container | 容器 |
| host | 本機、主機、宿主 |
| image | 映像檔 |
| registry | 倉庫註冊伺服器 |
| repository | 倉庫 |
| volume | 資料卷 |

---

## Install Docker

```bash
brew cask install docker
```

---

## Docker Introduction

### Basic Concept

執行 Docker 的主機稱為 Host，在 host 上執行 `docker run` 指令時，Docker 會操作以下三個元件來完成任務：

- **映像檔 (image)**：光碟片，唯讀且不能獨立執行
- **容器 (container)**：硬碟，可讀可寫可執行
- **倉庫 (repository)**：光碟盒的零售商

### Image

Image 包裝了一個執行特定環境所需要的資源。

每個 image 都有獨一無二的 digest，這是從 image 內容做 sha256 產生的。這個設計能讓 image 無法隨意改變內容，維持資料一致性。

雖然 image 裡有必要的資源，但它無法獨立執行，必須要靠 container 間接執行。

### Container

基於 image 可以建立出 Container。

它的概念像是建立一個可讀寫內容的外層，架在 image 上。實際存取 container 會經過可讀寫層與 image，因此看到的內容會是兩者合併後的結果。

Container 特性跟 image 不一樣，因為有可讀寫層，所以 container 可以讀寫，也可以拿來執行。

### Repository

Repository 是存放 image 的空間。

Docker 設計類似分散式版本控制的方法來存放各種 image。而分散式架構就會有類似 git 的 pull / push 行為，實際做的事也跟 git 類似：為了要跟遠端的 repository 同步。

另一個與 repository 很像，但容易混用的名詞為 **Registry**，後者涵蓋範圍更廣，包含了更多 repository 與身分驗證功能等，通常比較常討論的也是 registry。

目前 Docker 預設的 registry 為 [DockerHub](https://hub.docker.com/)，大多數程式或服務的 image 都能在上面找得到。
