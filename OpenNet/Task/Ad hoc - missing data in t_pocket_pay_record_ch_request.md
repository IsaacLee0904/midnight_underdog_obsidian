![[Screenshot 2026-03-25 at 10.51.33 AM.png]]

#### Pipeline Overview
![[afbet_pocket.t_pocket_pay_record_ch_request_shard|800]]

#### Troubleshooting
根據 [Flat table 文件](https://opennetltd.atlassian.net/wiki/spaces/BT/pages/2950758486/Flat+tables) afbet_pocket_{country code}.t_pocket_pay_record_ch_request 只會保留 90 天，而 user 提供的 pay_id 實際上是 12 月的紀錄已經超過 90 天因此會被刪除

![[Pasted image 20260325114021.png]]


