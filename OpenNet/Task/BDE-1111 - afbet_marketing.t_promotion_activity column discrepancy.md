## Information

* **Jira Ticket** : [Link](https://opennetltd.atlassian.net/jira/software/projects/BDE/boards/117?jql=assignee%20%3D%20712020%3A993acc76-02e5-4173-9e2b-61373ff1764c&selectedIssue=BDE-1111)
* **Branch** : <span style="color:rgb(8, 186, 118)">feature/BDE-1111_add_new_col_to_t_promotion_activity</span>


![[Screenshot 2026-03-31 at 2.15.31 PM.png]]

在 RDS 中有 `promotion_title_cms_page` 與 `promotion_title_cms_key` 兩個 columns 但 Redshift 沒有

**Step1. Check in RDS Prod and UAT**

- [x] Prod Sporty
- [x] UAT Sporty
- [ ] Prod Encore
- [ ] UAT Encore

after investigate only Sporty had added two new column

**Step2. Add columns with dba-tools**
Prod
```sql
ALTER TABLE afbet_marketing_gh.t_promotion_activity ADD COLUMN promotion_title_cms_page VARCHAR(64);
ALTER TABLE afbet_marketing_ng.t_promotion_activity ADD COLUMN promotion_title_cms_page VARCHAR(64);
ALTER TABLE afbet_marketing_int.t_promotion_activity ADD COLUMN promotion_title_cms_page VARCHAR(64);
ALTER TABLE afbet_marketing_br.t_promotion_activity ADD COLUMN promotion_title_cms_page VARCHAR(64);
ALTER TABLE afbet_marketing_ke.t_promotion_activity ADD COLUMN promotion_title_cms_page VARCHAR(64);
ALTER TABLE afbet_marketing_tz.t_promotion_activity ADD COLUMN promotion_title_cms_page VARCHAR(64);
ALTER TABLE afbet_marketing_za.t_promotion_activity ADD COLUMN promotion_title_cms_page VARCHAR(64);
ALTER TABLE afbet_marketing_zm.t_promotion_activity ADD COLUMN promotion_title_cms_page VARCHAR(64);

  
ALTER TABLE afbet_marketing_gh.t_promotion_activity ADD COLUMN promotion_title_cms_key VARCHAR(64);
ALTER TABLE afbet_marketing_ng.t_promotion_activity ADD COLUMN promotion_title_cms_key VARCHAR(64);
ALTER TABLE afbet_marketing_int.t_promotion_activity ADD COLUMN promotion_title_cms_key VARCHAR(64);
ALTER TABLE afbet_marketing_br.t_promotion_activity ADD COLUMN promotion_title_cms_key VARCHAR(64);
ALTER TABLE afbet_marketing_ke.t_promotion_activity ADD COLUMN promotion_title_cms_key VARCHAR(64);
ALTER TABLE afbet_marketing_tz.t_promotion_activity ADD COLUMN promotion_title_cms_key VARCHAR(64);
ALTER TABLE afbet_marketing_za.t_promotion_activity ADD COLUMN promotion_title_cms_key VARCHAR(64);
ALTER TABLE afbet_marketing_zm.t_promotion_activity ADD COLUMN promotion_title_cms_key VARCHAR(64);
```

UAT
```sql
ALTER TABLE afbet_marketing_gh.t_promotion_activity ADD COLUMN promotion_title_cms_page VARCHAR(64);
ALTER TABLE afbet_marketing_ng.t_promotion_activity ADD COLUMN promotion_title_cms_page VARCHAR(64);
ALTER TABLE afbet_marketing_int.t_promotion_activity ADD COLUMN promotion_title_cms_page VARCHAR(64);

ALTER TABLE afbet_marketing_gh.t_promotion_activity ADD COLUMN promotion_title_cms_key VARCHAR(64);
ALTER TABLE afbet_marketing_ng.t_promotion_activity ADD COLUMN promotion_title_cms_key VARCHAR(64);
ALTER TABLE afbet_marketing_int.t_promotion_activity ADD COLUMN promotion_title_cms_key VARCHAR(64);
```




