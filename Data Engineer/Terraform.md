# Terraform

tags: #terraform #iac #devops #data-engineering
source: [[Isaac's Note]]

---

## Terraform Fundamental

### What is Terraform

Terraform 是一套由 **HashiCorp** 開發的開源雲端部署工具 (Infrastructure as Code / IaC)，使用 **GO語言** 編寫。可以幫助我們自動化管理和部署基礎架構與雲端資源。

透過 Terraform，不再需要手動到 Web 介面開啟一台機器或手動設定 VPC，只要撰寫好 Terraform 腳本，一個鍵就可以達到你想要做的事情：

- **定義檔案設定**：透過定義檔案自動化設定和管理基礎架構，使其能夠輕鬆地複製和部署雲端元件
- **支持多個供應商**：支持多個雲服務供應商，如 AWS、Azure、Google Cloud、Oracle 等，可達到雲端通用的目標
- **簡單易懂的配置語言 HCL**：提供簡單易懂的配置語言 HCL（HashiCorp 配置語言），使用者更容易上手
- **可控制的基礎架構**：確保不會意外更改現有的基礎架構配置

### Install Terraform

```bash
# Install the HashiCorp tap
brew tap hashicorp/tap

# Install Terraform
brew install hashicorp/tap/terraform

# Update
brew update
brew upgrade hashicorp/tap/terraform
```

---

## Terraform 的元件

### Terraform Workflow

1. **Terraform init**：初始化本地 Terraform 環境
2. **Terraform fmt / validate**：格式化代碼與驗證語法及結構
3. **Terraform plan**：比較 Terraform 狀態和雲端中實際狀態，建立執行計畫（不會實際執行）
4. **Terraform apply**：根據計畫執行實際的基礎架構創建或更改操作
5. **Terraform destroy**：刪除此特定 Terraform 環境所管理的所有資源

執行位置：
- init / fmt / validate / plan → 在**本地**電腦完成
- apply / destroy → 在**雲端/地端架構**中實際執行

### Terraform Core Component

1. **安裝執行檔 (Executable)**：Terraform 的執行檔，用來與基礎設施進行溝通與執行
2. **雲端架構套件 (Provider plugins)**：提供簡化基礎架構之後的元件，如 AWS、GCP、Azure 等。核心元件：**Provider（供應商）**、**Module（模組）**
3. **元件配置檔 (Configuration File)**：定義與描述想要控制的基礎架構。核心元件：**Resources（資源）**、**Variables（變數）**、**Data Sources（資料來源）**、**Outputs（輸出）**
4. **狀態檔案 (State Data)**：管理實際基礎設施的狀態和資源與程式碼之間的關係。核心元件：**State（狀態）**、**Backend（遠端狀態後台）**

### Terraform Component Hierarchy

- **藍色部分（Provider）**：與雲端供應商的元件配置有關，支援 AWS、Azure、GCP 等
- **綠色部分（Module）**：將雲端元件抽象化的可重複使用 Terraform 代碼塊
- **紫色部分（HCL 元件）**：
  1. Provisioners（配置器）：在資源創建後設置它們的狀態
  2. Resources（資源）：代表基礎架構中的一個單一元件
  3. Variables（變數）：用於設置 Terraform 模組的參數
  4. Data Sources（資料來源）：從現有的基礎架構中檢索資料
  5. Outputs（輸出）：顯示 Terraform 模組創建的基礎架構元件的屬性
- **黃色部分（State）**：
  1. State（狀態）：記錄當前基礎架構的狀態
  2. Backend（遠端狀態後台）：存儲 Terraform 狀態的後端

---

## Terraform Setup

### Configuration

```yaml
provider "google" {
  access_key = "ACCESS_KEY_HERE"
  secret_key = "SECRET_KEY_HERE"
  region     = "ap-northeast-1"
}

# resource <resource_type> <resource_name>
resource "aws_instance" "example" {
  ami           = "ami-0ccdbc8c1cb7957be"
  instance_type = "t2.micro"
}
```

- `provider`：指定所要使用的 provider（aws、gcp 等）
- `resource`：定義要建立的資源
- `<resource_type>`：每個 provider 底下的不同資源類型（如 `aws_instance`）
- `<resource_name>`：按照個人喜好或需求定義

### Initialization

```bash
terraform init
```

執行後 Terraform 會下載相對應的 provider binary，並放到 `.terraform` 目錄中。
