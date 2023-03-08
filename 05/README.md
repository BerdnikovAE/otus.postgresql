## ДЗ 05. Настройка autovacuum с учетом оптимальной производительности

``` bash
# в терминале GOOGLE CLOUD SHELL делаем VM
gcloud compute instances create vm-postgresql \
    --project postgresql-linux-2023 \
    --zone=europe-north1-a \
    --machine-type=e2-medium \
    --create-disk=auto-delete=yes,boot=yes,image=projects/ubuntu-os-cloud/global/images/ubuntu-2004-focal-v20230213,mode=rw,size=10 

# заходим в новую VM
PS C:\p\otus.postgresql.code> ssh -i gcp ae@34.88.169.125

# ставим postgresql 14-ый
sudo apt update && sudo apt upgrade -y && sudo sh -c 'echo "deb http://apt.postgresql.org/pub/repos/apt $(lsb_release -cs)-pgdg main" > /etc/apt/sources.list.d/pgdg.list' && wget --quiet -O - https://www.postgresql.org/media/keys/ACCC4CF8.asc | sudo apt-key add - && sudo apt-get update && sudo apt-get -y install postgresql-14

# параметры в конфиг
sudu su
echo '
max_connections = 40
shared_buffers = 1GB
effective_cache_size = 3GB
maintenance_work_mem = 512MB
checkpoint_completion_target = 0.9
wal_buffers = 16MB
default_statistics_target = 500
random_page_cost = 4
effective_io_concurrency = 2
work_mem = 6553kB
min_wal_size = 4GB
max_wal_size = 16GB
' | tee -a /etc/postgresql/14/main/postgresql.conf

# restart
systemctl stop postgresql && systemctl start postgresql

# запускаем тест
sudo -u postgres pgbench -i postgres
sudo -u postgres  pgbench -c8 -P 60 -T 600 -U postgres postgres

pgbench (14.7 (Ubuntu 14.7-1.pgdg20.04+1))
starting vacuum...end.
progress: 60.0 s, 698.7 tps, lat 11.440 ms stddev 6.972
progress: 120.0 s, 707.4 tps, lat 11.307 ms stddev 6.820
progress: 180.0 s, 699.7 tps, lat 11.431 ms stddev 7.328
progress: 240.0 s, 665.1 tps, lat 12.026 ms stddev 7.864
progress: 300.0 s, 714.1 tps, lat 11.202 ms stddev 6.943
progress: 360.0 s, 568.9 tps, lat 14.058 ms stddev 17.538
progress: 420.0 s, 455.1 tps, lat 17.574 ms stddev 24.920
progress: 480.0 s, 449.2 tps, lat 17.797 ms stddev 25.321
progress: 540.0 s, 454.2 tps, lat 17.606 ms stddev 24.641
progress: 600.0 s, 439.3 tps, lat 18.208 ms stddev 25.077
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 8
number of threads: 1
duration: 600 s
number of transactions actually processed: 351110
latency average = 13.667 ms
latency stddev = 16.146 ms
initial connection time = 37.166 ms
tps = 585.193061 (without initial connection time)
# видно, что после 5 минут ему заплохело
# это было на default настройках 
postgres=# select name, setting from pg_settings where name like '%vacuum%';
                 name                  |  setting
---------------------------------------+------------
 autovacuum                            | on
 autovacuum_analyze_scale_factor       | 0.1
 autovacuum_analyze_threshold          | 50
 autovacuum_freeze_max_age             | 200000000
 autovacuum_max_workers                | 3
 autovacuum_multixact_freeze_max_age   | 400000000
 autovacuum_naptime                    | 60
 autovacuum_vacuum_cost_delay          | 2
 autovacuum_vacuum_cost_limit          | -1
 autovacuum_vacuum_insert_scale_factor | 0.2
 autovacuum_vacuum_insert_threshold    | 1000
 autovacuum_vacuum_scale_factor        | 0.2
 autovacuum_vacuum_threshold           | 50
 autovacuum_work_mem                   | -1
 log_autovacuum_min_duration           | -1
 vacuum_cost_delay                     | 0
 vacuum_cost_limit                     | 200
 vacuum_cost_page_dirty                | 20
 vacuum_cost_page_hit                  | 1
 vacuum_cost_page_miss                 | 2
 vacuum_defer_cleanup_age              | 0
 vacuum_failsafe_age                   | 1600000000
 vacuum_freeze_min_age                 | 50000000
 vacuum_freeze_table_age               | 150000000
 vacuum_multixact_failsafe_age         | 1600000000
 vacuum_multixact_freeze_min_age       | 5000000
 vacuum_multixact_freeze_table_age     | 150000000
(27 rows)

# теперь меняем ключевые для autovacuum параметры 
echo '
log_autovacuum_min_duration = 0
autovacuum_max_workers = 10
autovacuum_naptime = 15s
autovacuum_vacuum_threshold = 25
autovacuum_vacuum_scale_factor = 0.05
autovacuum_vacuum_cost_delay = 10
autovacuum_vacuum_cost_limit = 1000
' | tee -a /etc/postgresql/14/main/postgresql.conf

# снова запускаем после ребута 
systemctl stop postgresql && systemctl start postgresql

sudo -u postgres  pgbench -c8 -P 60 -T 600 -U postgres postgres
pgbench (14.7 (Ubuntu 14.7-1.pgdg20.04+1))
starting vacuum...end.
progress: 60.0 s, 673.0 tps, lat 11.878 ms stddev 6.877
progress: 120.0 s, 666.8 tps, lat 11.996 ms stddev 6.701
progress: 180.0 s, 684.4 tps, lat 11.688 ms stddev 6.530
progress: 240.0 s, 690.8 tps, lat 11.579 ms stddev 6.323
progress: 300.0 s, 667.7 tps, lat 11.979 ms stddev 6.815
progress: 360.0 s, 636.2 tps, lat 12.572 ms stddev 11.518
progress: 420.0 s, 424.1 tps, lat 18.861 ms stddev 25.169
progress: 480.0 s, 421.8 tps, lat 18.963 ms stddev 25.339
progress: 540.0 s, 426.4 tps, lat 18.759 ms stddev 25.191
progress: 600.0 s, 439.9 tps, lat 18.178 ms stddev 25.302
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 8
number of threads: 1
duration: 600 s
number of transactions actually processed: 343886
latency average = 13.956 ms
latency stddev = 15.522 ms
initial connection time = 30.425 ms
tps = 573.143969 (without initial connection time)
# вообще ничего не поменялось 

```

Что я делаею не так ? 
