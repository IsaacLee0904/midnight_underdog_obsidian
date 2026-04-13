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
在 config/role/ 底下建立以目標角色名稱命名的資料夾，命名為 <font color="#548dd4">config/role/{team}_ group</font> 或 <font color="#548dd4">config/role/{application_account_name}</font>，在 folder 下創建各自環境的 yaml file，發 PR 請人 approve
### Serverless vs Provisioned

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

