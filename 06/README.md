## ДЗ 06. Работа с журналами


```sh
# 1. делаем настройки
echo '
checkpoint_timeout = 30s
checkpoint_completion_target = 0.9
log_checkpoints = on 
' | tee -a /var/lib/postgresql/14/main/postgresql.auto.conf
pg_ctlcluster 14 main restart

# запомним номер lsn
postgres=# SELECT pg_current_wal_insert_lsn();
 pg_current_wal_insert_lsn
---------------------------
 2/77177FB8

postgres=# SELECT * FROM pg_stat_bgwriter \gx
-[ RECORD 1 ]---------+------------------------------
checkpoints_timed     | 93
checkpoints_req       | 40
checkpoint_write_time | 2062301
checkpoint_sync_time  | 5836
buffers_checkpoint    | 833844
buffers_clean         | 39099
maxwritten_clean      | 334
buffers_backend       | 495755
buffers_backend_fsync | 0
buffers_alloc         | 896497
stats_reset           | 2023-03-11 08:46:40.011262+00


# 2. запускаем тест
sudo -u postgres pgbench -i postgres
sudo -u postgres  pgbench -c8 -P 60 -T 600 -U postgres postgres

# снимаем статистику 

postgres=# SELECT pg_current_wal_insert_lsn();
 pg_current_wal_insert_lsn
---------------------------
 2/95764720

postgres=# SELECT * FROM pg_stat_bgwriter \gx
-[ RECORD 1 ]---------+------------------------------
checkpoints_timed     | 116
checkpoints_req       | 40
checkpoint_write_time | 2628043
checkpoint_sync_time  | 6026
buffers_checkpoint    | 876576
buffers_clean         | 39099
maxwritten_clean      | 334
buffers_backend       | 500305
buffers_backend_fsync | 0
buffers_alloc         | 901475
stats_reset           | 2023-03-11 08:46:40.011262+00

# 3. смотрим объем изменений
postgres=# select pg_size_pretty(('2/95764720'::pg_lsn-'2/77177FB8'::pg_lsn));
 pg_size_pretty
----------------
 486 MB
(1 row)
# = 24 MB за checkpoint

# 4. похоже, что все контрольные точки выполнялись по расписанию:
# приросло толко checkpoints_timed (+23) по расписанию 
# checkpoints_req по требованию не изменилось 

# да и в логах такая картина
2023-03-11 14:59:45.166 UTC [24593] LOG:  checkpoint starting: time
2023-03-11 15:00:12.156 UTC [24593] LOG:  checkpoint complete: wrote 1949 buffers (11.9%); 0 WAL file(s) added, 0 removed, 1 recycled; write=26.963 s, sync=0.006 s, total=26.990 s; sync files=10, longest=0.005 s, average=0.001 s; distance=21588 kB, estimate=25377 kB
2023-03-11 15:00:15.159 UTC [24593] LOG:  checkpoint starting: time
2023-03-11 15:00:42.159 UTC [24593] LOG:  checkpoint complete: wrote 1947 buffers (11.9%); 0 WAL file(s) added, 0 removed, 1 recycled; write=26.971 s, sync=0.009 s, total=27.000 s; sync files=14, longest=0.005 s, average=0.001 s; distance=21597 kB, estimate=24999 kB
2023-03-11 15:00:45.159 UTC [24593] LOG:  checkpoint starting: time
2023-03-11 15:01:12.161 UTC [24593] LOG:  checkpoint complete: wrote 1897 buffers (11.6%); 0 WAL file(s) added, 0 removed, 2 recycled; write=26.980 s, sync=0.006 s, total=27.002 s; sync files=9, longest=0.005 s, average=0.001 s; distance=21554 kB, estimate=24654 kB
2023-03-11 15:01:15.163 UTC [24593] LOG:  checkpoint starting: time
2023-03-11 15:01:42.160 UTC [24593] LOG:  checkpoint complete: wrote 2185 buffers (13.3%); 0 WAL file(s) added, 0 removed, 1 recycled; write=26.976 s, sync=0.011 s, total=26.997 s; sync files=16, longest=0.004 s, average=0.001 s; distance=21437 kB, estimate=24333 kB

# делаем (для интереса, вне домашки) настройки так чтоб чекпоинт требовалось по объему 
echo '
checkpoint_timeout = 60s
max_wal_size = 32MB
checkpoint_completion_target = 0.9
log_checkpoints = on 
' | tee -a /var/lib/postgresql/14/main/postgresql.auto.conf
pg_ctlcluster 14 main restart

# помогло, теперь есть checkpoints_req
postgres=# SELECT * FROM pg_stat_bgwriter \gx
-[ RECORD 1 ]---------+------------------------------
checkpoints_timed     | 377
checkpoints_req       | 86
checkpoint_write_time | 3124593
checkpoint_sync_time  | 10501
buffers_checkpoint    | 996820
buffers_clean         | 72664
maxwritten_clean      | 642
buffers_backend       | 608568
buffers_backend_fsync | 0
buffers_alloc         | 980031
stats_reset           | 2023-03-11 08:46:40.011262+00


# 5. Сравниваем синхронный и асинхронный режим 
postgres=# alter system reset all;

## Синхронный режим 
postgres=# show synchronous_commit;
 synchronous_commit
--------------------
 on
root@vm-postgresql:/home/ae# sudo -u postgres  pgbench -c8 -P 5 -T 30 -U postgres postgres
pgbench (14.7 (Ubuntu 14.7-1.pgdg20.04+1))
starting vacuum...end.
progress: 5.0 s, 916.8 tps, lat 8.668 ms stddev 5.264
progress: 10.0 s, 907.0 tps, lat 8.815 ms stddev 5.669
progress: 15.0 s, 919.0 tps, lat 8.708 ms stddev 5.498
progress: 20.0 s, 931.8 tps, lat 8.585 ms stddev 5.660
progress: 25.0 s, 925.2 tps, lat 8.646 ms stddev 5.622
progress: 30.0 s, 926.8 tps, lat 8.632 ms stddev 5.382
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 8
number of threads: 1
duration: 30 s
number of transactions actually processed: 27641
latency average = 8.677 ms
latency stddev = 5.521 ms
initial connection time = 27.816 ms
tps = 921.566616 (without initial connection time)

postgres=# alter system set synchronous_commit=off;
ALTER SYSTEM
## Асинхронный, в два раза веселей
root@vm-postgresql:/home/ae# sudo -u postgres  pgbench -c8 -P 5 -T 30 -U postgres postgres
pgbench (14.7 (Ubuntu 14.7-1.pgdg20.04+1))
starting vacuum...end.
progress: 5.0 s, 1819.2 tps, lat 4.371 ms stddev 1.374
progress: 10.0 s, 1870.6 tps, lat 4.276 ms stddev 1.283
progress: 15.0 s, 1848.8 tps, lat 4.326 ms stddev 1.208
progress: 20.0 s, 1807.4 tps, lat 4.426 ms stddev 1.351
progress: 25.0 s, 1828.0 tps, lat 4.375 ms stddev 1.295
progress: 30.0 s, 1822.6 tps, lat 4.388 ms stddev 1.295
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 8
number of threads: 1
duration: 30 s
number of transactions actually processed: 54991
latency average = 4.361 ms
latency stddev = 1.305 ms
initial connection time = 26.538 ms
tps = 1833.476729 (without initial connection time)

# 6. Пробуем контрольные суммы
# запомним табличку
postgres=# SELECT pg_relation_filepath('t');
 pg_relation_filepath
----------------------
 base/13726/16707
(1 row)

# включаем на живом кластере 
root@vm-postgresql:/home/ae# pg_ctlcluster 14 main stop
root@vm-postgresql:/home/ae# /usr/lib/postgresql/14/bin/pg_checksums -D/var/lib/postgresql/14/main -e
Checksum operation completed
Files scanned:  947
Blocks scanned: 5646
pg_checksums: syncing data directory
pg_checksums: updating control file
Checksums enabled in cluster

# портим файл 
root@vm-postgresql:/var/lib/postgresql/14/main/base/13726# dd if=/dev/zero of=/var/lib/postgresql/14/main/base/13726/16707 oflag=dsync conv=notrunc bs=1 count=8
8+0 records in
8+0 records out
8 bytes copied, 0.00538128 s, 1.5 kB/s
root@vm-postgresql:/home/ae# pg_ctlcluster 14 main start

# сломалось
postgres=# SELECT * from t;
WARNING:  page verification failed, calculated checksum 41687 but expected 63634
ERROR:  invalid page in block 0 of relation base/13726/16707

# игнорируем
postgres=# alter system set ignore_checksum_failure = on;
ALTER SYSTEM
postgres=# SELECT pg_reload_conf();
 pg_reload_conf
----------------
 t
(1 row)

postgres=# SELECT * from t;
WARNING:  page verification failed, calculated checksum 41687 but expected 63634
 t
----
 12
 12
(2 rows)
# прочиталось 







```