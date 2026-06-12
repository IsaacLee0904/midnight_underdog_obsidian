有時候會有有非 BI-users 想要取得 BI 的資料查詢權限，這樣才可以自己解決一些 Ad-hoc，目前的做法是透過 metabase 讓他們查詢 ([docs](https://opennetltd.atlassian.net/wiki/spaces/DET/pages/4473421881/Granting+Restricted+Database+Access+to+Non-BI+Users+via+Metabase))，但隨著這樣的需求增加，我們開始意識到到底哪些資料可以被 non-BI users 查詢哪些不行 ([thread](https://opennetltd.slack.com/archives/C07347VTBB2/p1781089134543059))![[Screenshot 2026-06-12 at 10.26.46 AM.png|546]]![[Screenshot 2026-06-12 at 10.27.14 AM.png|543]]

針對這個原因 我覺得我有點想將 [DAMA](https://github.com/batermj/data-analysis-1/blob/master/%E6%95%B0%E6%8D%AE%E5%88%86%E6%9E%90/DAMA%20%E6%95%B0%E6%8D%AE%E7%AE%A1%E7%90%86%E7%9F%A5%E8%AF%86%E4%BD%93%E7%B3%BB%E6%8C%87%E5%8D%97%EF%BC%88%E4%B8%AD%E6%96%87%E7%89%88%EF%BC%89.pdf) 的框架導入，然後透過在 OM 上面 labeling 資料分級來看是不是可以提供給 user

[DAMA-DMBOK]

(https://static1.squarespace.com/static/5530dddfe4b0679504639dc1/t/6773ae97cb4ccf79b34a711a/1735634600737/DAMA+Guide+1.pdf)
![[Screenshot 2026-06-12 at 11.10.56 AM.png]]

1. 要怎麼分級 >> 待討論
2. 