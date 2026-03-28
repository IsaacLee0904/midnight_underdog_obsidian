# Ansible

tags: #ansible #devops #automation #data-engineering
source: [[Isaac's Note]]

---

## Ansible Fundamental

### What is Ansible

`Ansible` 是適用於 Python 環境的自動化組態管理工具。組態（Configuration Management）意味著環境的設定及部署，使用 Ansible 可以將「制式化的系統操作」自動化，讓開發者將注意力投注在值得關注的事物上。

撰寫好「部署的流程」（例如安裝套件、複製檔案）及「遠端主機的連線資訊」後，Ansible 即可連線至其他主機，進行自動部署。

### Why Ansible

**Ansible 的優勢**：
- 在遠端只需要 python 環境與 ssh server，不需要其他 agent，因此易於部署
- 它使用 YAML，以 Ansible Playbook 的形式，允許您以接近英語的方式描述您的自動化作業

**為什麼使用 Ansible**：
1. **提升 Production 環境的穩定性及可靠性**：使用檔案管理「部署內容」，達到 Infrastructure as Code，可以使用版本控制進行進退版操作
2. **減少服務中斷時間**：若發生服務中斷，Ansible 能按照「預先設計好的流程」重新部署
3. **確保環境間（開發、測試、正式）設定對齊**：統一部署，避免人為失誤

### Ansible Project Structure

```bash
ansible_project
|- README.md
|- INVENTORY_FILE_1  # 概念 - Inventory
|- INVENTORY_FILE_2
|- main.yml  # 概念 - Playbook
|
|- group_vars/
    |- GROUP1.yml
    |- GROUP2.yml
|- roles/
    |- common/
        |- tasks/
            |- main.yml
        |- handlers/
            |- main.yml
```

---

## Ansible Terms

### Playbooks

Playbook 用來設定工作的流程與步驟，精細地編排基礎設施：

- `Play`：定義了 hosts 該執行什麼樣的 task
- `Task`：具體的工作，可能是一個 bash 指令

```yaml
# main.yml
- name: The example playbook  # 階層1 - Play
  hosts: localhost             # 定義執行此 Play 的機器
  vars:
    dynamic_word: "Hello World"

  tasks:
  - name: generate the hello_world.txt file  # task-1
    lineinfile:
      path: /tmp/hello_world.txt
      state: present
      line: "{{ dynamic_word }}"
      create: yes

  - name: show file context  # task-2
    command: cat /tmp/hello_word.txt
    register: result

  - name: print file result  # task-3
    debug:
      msg: "{{ result.stdout_lines }}"
```

### Modules

透過 Ansible 推送到各個節點的腳本稱為 "module"，Ansible 預設會透過 ssh 來執行 module。許多相關的 Module 集合起來會變成 Collection。

### Plugins

用來擴充 Ansible 的核心功能，如轉換資料、記錄輸出、連接到 inventory 等。Plugin 主要執行在 Control Node，與 Module 不同的是 Module 處理遠端操作，Plugin 處理核心輸入輸出。

### Inventory

```yaml
[webservers]
www1.example.com
www2.example.com

[dbservers]
db0.example.com
db1.example.com
```

### Variables

在 Ansible 的專案資料夾中有兩種類型的 Variable：`Host Variable` 以及 `Group Variable`，透過兩種變數的交互使用，可以快速且彈性的管理多台機器。

### Ansible Search Path

Ansible 按以下方式自動載入 modules、Module utilities、plugins：

```yaml
# ansible.cfg 或環境變數中的特定目錄
DEFAULT_MODULE_PATH
DEFAULT_MODULE_UTILS_PATH
DEFAULT_CACHE_PLUGIN_PATH
```

Ansible 的設定會依序從以下路徑載入：

```yaml
ANSIBLE_CONFIG
./ansible.cfg
~/.ansible.cfg
/etc/ansible/ansible.cfg
```
