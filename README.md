# K/V对比

https://blog.csdn.net/huxinglixing/article/details/116156322

https://github.com/huxinhuxin/kvtest

```
一万

[./kvtest -f ./ -a 10000]
2023/06/28 11:31:23 begin test bolt !!!!!!!!!!!!!
.//bolt.db
2023/06/28 11:31:23 sequence write begin
2023/06/28 11:31:26 sequence write end, cost time  3.069157471s
2023/06/28 11:31:26 randRead  begin
2023/06/28 11:31:26 randRead err count =  0
2023/06/28 11:31:26 randdel  begin
2023/06/28 11:31:30 randdel err count =  0
2023/06/28 11:31:30 randdel  end, cost time  3.683270353s
2023/06/28 11:31:30 randwrite  begin
2023/06/28 11:31:34 randwrite  end, cost time  4.478815804s
2023/06/28 11:31:34 readall  begin
2023/06/28 11:31:34 readall get count =  7708
2023/06/28 11:31:34 readall  end, cost time  1.052078ms
2023/06/28 11:31:34 10000 counts sequence write, rand read,rand del,rand write,get all
2023/06/28 11:31:34 db close ,total cost time  11.312932804s
2023/06/28 11:31:34 end test bolt !!!!!!!!!!!!!
2023/06/28 11:31:34 begin test pebble !!!!!!!!!!!!!
.//pebble
2023/06/28 11:31:34 sequence write begin
2023/06/28 11:31:35 sequence write end, cost time  853.149329ms
2023/06/28 11:31:35 randRead  begin
2023/06/28 11:31:35 randRead err count =  0
2023/06/28 11:31:35 randdel  begin
2023/06/28 11:31:36 randdel err count =  0
2023/06/28 11:31:36 randdel  end, cost time  570.763802ms
2023/06/28 11:31:36 randwrite  begin
2023/06/28 11:31:37 randwrite  end, cost time  722.175197ms
2023/06/28 11:31:37 readall  begin
2023/06/28 11:31:37 readall get count =  7629
2023/06/28 11:31:37 readall  end, cost time  73.831722ms
2023/06/28 11:31:37 10000 counts sequence write, rand read,rand del,rand write,get all
2023/06/28 11:31:37 db close ,total cost time  2.327886891s
2023/06/28 11:31:37 end test pebble !!!!!!!!!!!!!
2023/06/28 11:31:37 begin test badger !!!!!!!!!!!!!
.//badger
badger 2023/06/28 11:31:37 INFO: All 0 tables opened in 0s
badger 2023/06/28 11:31:37 INFO: Discard stats nextEmptySlot: 0
badger 2023/06/28 11:31:37 INFO: Set nextTxnTs to 0
2023/06/28 11:31:37 sequence write begin
2023/06/28 11:31:41 sequence write end, cost time  3.79018787s
2023/06/28 11:31:41 randRead  begin
2023/06/28 11:31:41 randRead err count =  0
2023/06/28 11:31:41 randdel  begin
2023/06/28 11:31:42 randdel err count =  0
2023/06/28 11:31:42 randdel  end, cost time  1.198206989s
2023/06/28 11:31:42 randwrite  begin
2023/06/28 11:31:46 randwrite  end, cost time  3.77515386s
2023/06/28 11:31:46 readall  begin
2023/06/28 11:31:46 readall get count =  7725
2023/06/28 11:31:46 readall  end, cost time  16.04366ms
badger 2023/06/28 11:31:46 INFO: Lifetime L0 stalled for: 0s
badger 2023/06/28 11:31:46 INFO:
Level 0 [ ]: NumTables: 01. Size: 489 KiB of 0 B. Score: 0.00->0.00 Target FileSize: 64 MiB
Level 1 [ ]: NumTables: 00. Size: 0 B of 10 MiB. Score: 0.00->0.00 Target FileSize: 2.0 MiB
Level 2 [ ]: NumTables: 00. Size: 0 B of 10 MiB. Score: 0.00->0.00 Target FileSize: 2.0 MiB
Level 3 [ ]: NumTables: 00. Size: 0 B of 10 MiB. Score: 0.00->0.00 Target FileSize: 2.0 MiB
Level 4 [ ]: NumTables: 00. Size: 0 B of 10 MiB. Score: 0.00->0.00 Target FileSize: 2.0 MiB
Level 5 [ ]: NumTables: 00. Size: 0 B of 10 MiB. Score: 0.00->0.00 Target FileSize: 2.0 MiB
Level 6 [B]: NumTables: 00. Size: 0 B of 10 MiB. Score: 0.00->0.00 Target FileSize: 2.0 MiB
Level Done
2023/06/28 11:31:46 10000 counts sequence write, rand read,rand del,rand write,get all
2023/06/28 11:31:46 db close ,total cost time  8.88882326s
2023/06/28 11:31:46 end test badger !!!!!!!!!!!!!
--------------- bolt --------------

seque write cost time:        3.069157471s
rand read cost time:          75.199069ms
rand del cost time:           3.683270353s
rand write cost:              4.478815804s
getall cost:                  1.052078ms
total cost:                   11.312932804s

-----------------------------------
--------------- pebble --------------

seque write cost time:        853.149329ms
rand read cost time:          91.731766ms
rand del cost time:           570.763802ms
rand write cost:              722.175197ms
getall cost:                  73.831722ms
total cost:                   2.327886891s

-----------------------------------
--------------- badger --------------

seque write cost time:        3.79018787s
rand read cost time:          73.084294ms
rand del cost time:           1.198206989s
rand write cost:              3.77515386s
getall cost:                  16.04366ms
total cost:                   8.88882326s

-----------------------------------


10万
--------------- pebble --------------

seque write cost time:        6.341589103s
rand read cost time:          1.342146869s
rand del cost time:           5.2483614s
rand write cost:              11.333176547s
getall cost:                  692.895844ms
total cost:                   25.07367853s

-----------------------------------
--------------- badger --------------

seque write cost time:        35.472496258s
rand read cost time:          705.603311ms
rand del cost time:           9.636414299s
rand write cost:              35.15595596s
getall cost:                  146.364165ms
total cost:                   1m21.403964226s

-----------------------------------
--------------- bolt --------------

seque write cost time:        34.574638108s
rand read cost time:          914.090331ms
rand del cost time:           2m16.546609363s
rand write cost:              2m9.629712722s
getall cost:                  13.770647ms
total cost:                   5m1.710829044s

-----------------------------------
```

