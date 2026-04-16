![[Screenshot 2026-04-16 at 3.42.52 PM.png]]

`t_patron_user_meta`
- There was a exist pipeline (`sporty_patron.t_patron_user_meta_backfill.py`) will use it for backfill.
- target table : `sporty_patron_global.t_patron_user_meta``
- backfill start : min(create_time)
- dag_schedule : daily
- min(create_time) : 2020-09-11 10:02:56.504
- 20260416 11:14 start to backfill, ETA 20260418
- row_count : 9610323
    
`t_patron_sso_user_meta`
- target table : `sporty_patron_global.t_patron_sso_user_meta`
- backfill start : min(create_time)
- dag_schedule : daily
- min(create_time) : 2021-12-02 03:06:07.460150
- row_count : 23964481

其中 ``

