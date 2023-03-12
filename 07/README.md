## ДЗ 07. Механизм блокировок

``` sql
### 1. Настройте сервер так, чтобы в журнал сообщений сбрасывалась информация о блокировках, удерживаемых более 200 миллисекунд. Воспроизведите ситуацию, при которой в журнале появятся такие сообщения.
 
ALTER SYSTEM SET deadlock_timeout = 200ms;
ALTER SYSTEM SET log_lock_waits = on;
SELECT pg_reload_conf();

-- Приготовим себе таличку
postgres=# create table t (t int);
CREATE TABLE
postgres=# insert into t values (1),(2),(3);
INSERT 0 3

-- ТРАНЗАКЦИЯ 1
postgres=# BEGIN;
BEGIN
postgres=*# UPDATE t SET t=10 WHERE t=1;
UPDATE 1
postgres=*#


                            -- ТРАНЗАКЦИЯ 2
                            postgres=# BEGIN
                            postgres-# ;
                            BEGIN
                            postgres=*# UPDATE t SET t=100 WHERE t=1;
                            -- висим


tail -f /var/log/postgresql/postgresql-14-main.log

2023-03-12 12:21:02.764 UTC [22171] postgres@postgres LOG:  process 22171 still waiting for ShareLock on transaction 743 after 200.147 ms
2023-03-12 12:21:02.764 UTC [22171] postgres@postgres DETAIL:  Process holding the lock: 22060. Wait queue: 22171.
2023-03-12 12:21:02.764 UTC [22171] postgres@postgres CONTEXT:  while updating tuple (0,5) in relation "t"
2023-03-12 12:21:02.764 UTC [22171] postgres@postgres STATEMENT:  UPDATE t SET t=100 WHERE t=1;

-- ТРАНЗАКЦИЯ 1
postgres=*# ROLLBACK;
ROLLBACK
                            -- ТРАНЗАКЦИЯ 2
                            UPDATE 0
                            postgres=*# ROLLBACK;
                            ROLLBACK

### 2. Смоделируйте ситуацию обновления одной и той же строки тремя командами UPDATE в разных сеансах. Изучите возникшие блокировки в представлении pg_locks и убедитесь, что все они понятны. Пришлите список блокировок и объясните, что значит каждая.

-- ТРАНЗАКЦИЯ 1
postgres=# BEGIN;
BEGIN
postgres=*# SELECT txid_current(), pg_backend_pid();
 txid_current | pg_backend_pid
--------------+----------------
          753 |          22060
(1 row)

postgres=*# UPDATE t SET t=10 WHERE t=1;
UPDATE 1
postgres=*#

                            -- ТРАНЗАКЦИЯ 2
                            postgres=# BEGIN;
                            BEGIN
                            postgres=*# SELECT txid_current(), pg_backend_pid();
                            txid_current | pg_backend_pid
                            --------------+----------------
                                      754 |          22171
                            (1 row)

                            postgres=*# UPDATE t SET t=100 WHERE t=1;
                            -- висим

                                                        -- ТРАНЗАКЦИЯ 2
                                                        postgres=# BEGIN;
                                                        BEGIN
                                                        postgres=*# SELECT txid_current(), pg_backend_pid();
                                                        txid_current | pg_backend_pid
                                                        --------------+----------------
                                                                  755 |          23179
                                                        (1 row)

                                                        postgres=*# UPDATE t SET t=1000 WHERE t=1;
                                                        -- висим

-- ТРАНЗАКЦИЯ 1
postgres=*# SELECT pid, wait_event_type, wait_event, pg_blocking_pids(pid)
FROM pg_stat_activity WHERE pid in (22060,22171,23179)  order by pid;
  pid  | wait_event_type |  wait_event   | pg_blocking_pids
-------+-----------------+---------------+------------------
 22060 |                 |               | {}
 22171 | Lock            | transactionid | {22060}
 23179 | Lock            | tuple         | {22171}
(3 rows)

-- видно что 
-- №1 22060 выполняется и никого не ждет
-- №2 22171 ждет 22060
-- №3 23179 ждет tuple от 22171

-- Тут подробней:
postgres=*# select * from locks_v;
  pid  |   locktype    | lockid |       mode       | granted
-------+---------------+--------+------------------+---------
 22060 | relation      | t      | RowExclusiveLock | t

 22171 | relation      | t      | RowExclusiveLock | t
 22171 | transactionid | 753    | ShareLock        | f
 22171 | tuple         | t:7    | ExclusiveLock    | t

 23179 | relation      | t      | RowExclusiveLock | t
 23179 | tuple         | t:7    | ExclusiveLock    | f
 
 23179 | transactionid | 755    | ExclusiveLock    | t
 22171 | transactionid | 754    | ExclusiveLock    | t
 22060 | transactionid | 753    | ExclusiveLock    | t
(9 rows)

### 3. Воспроизведите взаимоблокировку трех транзакций. Можно ли разобраться в ситуации постфактум, изучая журнал сообщений?

-- ТРАНЗАКЦИЯ 1
postgres=# create table tt (id int, n int);
CREATE TABLE
postgres=#
postgres=#
postgres=# insert into tt values (1,10),(2,20),(3,30);
INSERT 0 3
postgres=# BEGIN;
BEGIN
postgres=*# UPDATE tt SET n = n - 1 WHERE id = 1;
UPDATE 1
postgres=*#
                            -- ТРАНЗАКЦИЯ 2
                            postgres=# BEGIN;
                            BEGIN
                            postgres=*# UPDATE tt SET n = n - 2 WHERE id=2;
                            UPDATE 1
                            postgres=*#
                                                         -- ТРАНЗАКЦИЯ 3
                                                        postgres=# BEGIN;
                                                        BEGIN
                                                        postgres=*# UPDATE tt SET n = n - 3 WHERE id=3;
                                                        UPDATE 1
                                                        postgres=*#
-- ТРАНЗАКЦИЯ 1
postgres=*# UPDATE tt SET n = n + 2 WHERE id=2;
-- висим
                            -- ТРАНЗАКЦИЯ 2
                            postgres=*# UPDATE tt SET n = n + 3 WHERE id=3;
                            -- висим
                                                        -- ТРАНЗАКЦИЯ 3
                                                        postgres=*# UPDATE tt SET n = n + 1 WHERE id=1;
                                                        ERROR:  deadlock detected
                                                        DETAIL:  Process 23179 waits for ShareLock on transaction 765; blocked by process 22060.
                                                        Process 22060 waits for ShareLock on transaction 766; blocked by process 22171.
                                                        Process 22171 waits for ShareLock on transaction 767; blocked by process 23179.
                                                        HINT:  See server log for query details.
                                                        CONTEXT:  while updating tuple (0,1) in relation "tt"
                                                        postgres=!#
                                                        -- ошибка

2023-03-12 13:14:16.690 UTC [22060] postgres@postgres LOG:  process 22060 still waiting for ShareLock on transaction 766 after 200.188 ms
2023-03-12 13:14:16.690 UTC [22060] postgres@postgres DETAIL:  Process holding the lock: 22171. Wait queue: 22060.
2023-03-12 13:14:16.690 UTC [22060] postgres@postgres CONTEXT:  while updating tuple (0,2) in relation "tt"
2023-03-12 13:14:16.690 UTC [22060] postgres@postgres STATEMENT:  UPDATE tt SET n = n + 2 WHERE id=2;
2023-03-12 13:14:34.392 UTC [22171] postgres@postgres LOG:  process 22171 still waiting for ShareLock on transaction 767 after 200.187 ms
2023-03-12 13:14:34.392 UTC [22171] postgres@postgres DETAIL:  Process holding the lock: 23179. Wait queue: 22171.
2023-03-12 13:14:34.392 UTC [22171] postgres@postgres CONTEXT:  while updating tuple (0,3) in relation "tt"
2023-03-12 13:14:34.392 UTC [22171] postgres@postgres STATEMENT:  UPDATE tt SET n = n + 3 WHERE id=3;
2023-03-12 13:14:45.545 UTC [23179] postgres@postgres LOG:  process 23179 detected deadlock while waiting for ShareLock on transaction 765 after 200.205 ms
2023-03-12 13:14:45.545 UTC [23179] postgres@postgres DETAIL:  Process holding the lock: 22060. Wait queue: .
2023-03-12 13:14:45.545 UTC [23179] postgres@postgres CONTEXT:  while updating tuple (0,1) in relation "tt"
2023-03-12 13:14:45.545 UTC [23179] postgres@postgres STATEMENT:  UPDATE tt SET n = n + 1 WHERE id=1;
2023-03-12 13:14:45.545 UTC [23179] postgres@postgres ERROR:  deadlock detected
2023-03-12 13:14:45.545 UTC [23179] postgres@postgres DETAIL:  Process 23179 waits for ShareLock on transaction 765; blocked by process 22060.
        Process 22060 waits for ShareLock on transaction 766; blocked by process 22171.
        Process 22171 waits for ShareLock on transaction 767; blocked by process 23179.
        Process 23179: UPDATE tt SET n = n + 1 WHERE id=1;
        Process 22060: UPDATE tt SET n = n + 2 WHERE id=2;
        Process 22171: UPDATE tt SET n = n + 3 WHERE id=3;
2023-03-12 13:14:45.545 UTC [23179] postgres@postgres HINT:  See server log for query details.
2023-03-12 13:14:45.545 UTC [23179] postgres@postgres CONTEXT:  while updating tuple (0,1) in relation "tt"
2023-03-12 13:14:45.545 UTC [23179] postgres@postgres STATEMENT:  UPDATE tt SET n = n + 1 WHERE id=1;
2023-03-12 13:14:45.546 UTC [22171] postgres@postgres LOG:  process 22171 acquired ShareLock on transaction 767 after 11354.513 ms
2023-03-12 13:14:45.546 UTC [22171] postgres@postgres CONTEXT:  while updating tuple (0,3) in relation "tt"
2023-03-12 13:14:45.546 UTC [22171] postgres@postgres STATEMENT:  UPDATE tt SET n = n + 3 WHERE id=3;

-- видно, что всё шло не плохо, 22060 ждал 22171, 22171 ждал 23179, 
-- но как только 23179 стал ждать 22060 - сразу "detected deadlock"
-- да и все три update нарисованы. 

### 4. Могут ли две транзакции, выполняющие единственную команду UPDATE одной и той же таблицы (без where), заблокировать друг друга?
-- теоретически два update-а которые идут построчно могут пересечься на одной строке.
-- это можно сэмулировать на курсорах идущих навстречу друг другу 

-- ТРАНЗАКЦИЯ 1
postgres=# select * from tt;
 id | n
----+----
  1 | 10
  2 | 20
  3 | 30
(3 rows)

postgres=# BEGIN;
BEGIN
postgres=*# DECLARE c CURSOR FOR SELECT * FROM tt ORDER BY id FOR UPDATE;
DECLARE CURSOR

                                    -- ТРАНЗАКЦИЯ 2
                                    postgres=# BEGIN;
                                    BEGIN
                                    postgres=*# DECLARE c CURSOR FOR SELECT * FROM tt ORDER BY id DESC FOR UPDATE;
                                    DECLARE CURSOR
                                    postgres=*# FETCH c;
                                    id | n
                                    ----+----
                                    3 | 30
                                    (1 row)
-- ТРАНЗАКЦИЯ 1
postgres=*# FETCH C;
 id | n
----+----
  1 | 10
(1 row)

postgres=*# FETCH C;
 id | n
----+----
  2 | 20
(1 row)

                                    -- ТРАНЗАКЦИЯ 2
                                    postgres=*# FETCH C;
                                    --висим

-- ТРАНЗАКЦИЯ 1                                   
postgres=*# FETCH C;
ERROR:  deadlock detected
DETAIL:  Process 24929 waits for ShareLock on transaction 791; blocked by process 24933.
Process 24933 waits for ShareLock on transaction 792; blocked by process 24929.
HINT:  See server log for query details.
CONTEXT:  while locking tuple (0,3) in relation "tt"
postgres=!#

-- сломали!


```