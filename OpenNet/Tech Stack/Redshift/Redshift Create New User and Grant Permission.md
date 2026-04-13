## Introduction

對 UAT / Prod 的 Redshift 創建 user 或給予權限是使用 [dba-redshift-privileges](https://github.com/opennetltd/dba-redshift-privileges)  的 repo，repo 主要分成 <font color="#ff0000">user_account</font> 與 <font color="#ff0000">role</font> 兩部分需要定義，前者用來定義與創建 Redshift 的 user，後者則是定義要給予各個 user 哪些權限與 table 的許可
### user_account
在 <font color="#548dd4">config/user/ </font>的 folder 底下創建<font color="#ff0000">與帳號名稱一樣的 folder</font>  EX. app_metabase_trading ，在此之下再創建 UAT 與 Prod 兩個環境的 yaml file ( 可以參考 <font color="#548dd4">config/user/tempalte.yaml</font> )
user config 裡面的 permissions 欄位要填寫的 role 名稱需要去 reference config/role/ folder 底下已經存在的 role，並且命名規則如下：
1. user account：通常指派 role {team}_ group EX. role de_group, role da_group
2. application account：通常指派同名的 role EX. role app_pocket, role app_order

>[!WARNING] 如果 role 還不存在，就不能直接在 user config 裡指派它，要先建好 role 跑過 pipeline 才行
>

### role
在 config/role/ 底下建立以目標角色名稱命名的資料夾，命名為 <font color="#548dd4">config/role/{team}_ group</font> 或 <font color="#548dd4">config/role/{application_account_name}</font>


  使用者帳號：

  - 在 config/user/ 底下建立以目標帳號名稱命名的資料夾，例如 config/user/<firstname_lastname>/ 或 config/user/<application_account_name>/
  - 在該資料夾下建立環境設定檔 <env>.yaml，例如 config/user/john_doe/prod.yaml 或 config/user/app_pocket/prod.yaml
  - 複製 config/user/template.yaml 的內容作為範本。
  - 依照檔案說明填入所有資訊。
    - 目標權限角色請參考 config/role/ 底下的角色與權限設定，使用者的目標角色通常為 ROLE <team>_group 或 ROLE <application_account_name>。
    - 若目標角色尚未存在，請參考下方建立角色的流程，並與 database team 討論。
  - 發起 pull request 並取得主管 approve。
  - 將 pull request 分享給 database team。
  - Merge pull request 後，使用以下 pipeline 套用變更：Run Grants workflow。

  角色（Role）：

  - 在 config/role/ 底下建立以目標角色名稱命名的資料夾，命名規則為：config/role/<team>_group 或 config/role/<application_account_name>。
  - 在該資料夾下建立環境設定檔 <env>.yaml，例如 config/role/de_group/prod.yaml 或 config/role/app_pocket/prod.yaml
  - 複製 config/role/template.yaml 的內容作為範本。
  - 依照檔案說明填入所有資訊。
  - 發起 pull request 並取得主管 approve。
  - 將 pull request 分享給 database team。
  - Merge pull request 後，使用以下 pipeline 套用變更：Run Grants workflow。

  Serverless vs Provisioned

  Provisioned：
  - cluster_type：provisioned（省略時的預設值）
  - 必填欄位：cluster_id、port、secret（admin 使用者的 secret）

  Serverless：
  - cluster_type：serverless
  - workgroup：<workgroup_name>
  - database：選填（預設為 dev）
  - use_temp_credentials：
    - true（預設）→ 呼叫 redshift-serverless:GetCredentials（不需要 secret）
    - false → 透過 Secrets Manager 取得 secret（使用已存儲的 DB 使用者）
  - secret：僅在 use_temp_credentials = false 時需要
  - 只有 redshift-serverless:GetWorkgroup / GetCredentials 會使用假設角色（redshift.py 中的 SERVERLESS_ASSUME_ROLE_ARN 常數）。
  - Secrets Manager 存取一律使用預設的 runner / instance role。 


## Example
![[BDE-1147 - Create app account for trading access through metabase]]

