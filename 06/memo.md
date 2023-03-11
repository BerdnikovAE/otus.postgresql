## буферный кэш = shared buffers

``` sql
SHOW shared_buffers; -- в мегабайтах
ALTER SYSTEM SET shared_buffers=131072; -- в 8кБ страницах = 1GB
SELECT setting FROM pg_settings WHERE name='shared_buffers'; -- в страницах
# pg_ctlcluster 14 main restart

CREATE EXTENSION pg_buffercache;

-- удобная вьюшка
CREATE VIEW pg_buffercache_v AS
SELECT bufferid,
       (SELECT c.relname
        FROM   pg_class c
        WHERE  pg_relation_filenode(c.oid) = b.relfilenode
       ) relname,
       CASE relforknumber
         WHEN 0 THEN 'main'
         WHEN 1 THEN 'fsm'
         WHEN 2 THEN 'vm'
       END relfork,
       relblocknumber,
       isdirty,
       usagecount
FROM   pg_buffercache b
WHERE  b.reldatabase IN (
         0, (SELECT oid FROM pg_database WHERE datname = current_database())
       )
AND    b.usagecount IS NOT NULL;

-- вставим 1 млн строк 
CREATE TABLE t (t int);
INSERT INTO  t SELECT a FROM  generate_series(1,1000000) as s(a);

-- посмотрим а что там в буферах 
SELECT * FROM pg_buffercache_v where relname='t';

-- свободные буфера 
SELECT count(*) FROM pg_buffercache WHERE usagecount IS NULL;

-- сколько буферов с каким счетчиком использования и сколько там unused
SELECT usagecount, count(*)
FROM pg_buffercache
GROUP BY usagecount
ORDER BY usagecount;

-- сколько буферов чем заняты 
SELECT c.relname,
  count(*) blocks,
  round( 100.0 * 8192 * count(*) / pg_table_size(c.oid) ) "% of rel",
  round( 100.0 * 8192 * count(*) FILTER (WHERE b.usagecount > 3) / pg_table_size(c.oid) ) "% hot"
FROM pg_buffercache b
  JOIN pg_class c ON pg_relation_filenode(c.oid) = b.relfilenode
WHERE  b.reldatabase IN (
         0, (SELECT oid FROM pg_database WHERE datname = current_database())
       )
AND    b.usagecount is not null
GROUP BY c.relname, c.oid
ORDER BY 2 DESC
LIMIT 10;


-- очистки кеша не бывает, только через рестарт 

-- прогрев кеша
CREATE EXTENSION pg_prewarm;
SELECT pg_prewarm('test');

-- добавить в загрузку и тогда после ребута кэш будет восстанавливаться 
# postgresql.conf
shared_preload_libraries = 'pg_prewarm'
pg_prewarm.autoprewarm = true
pg_prewarm.autoprewarm_interval = 300s

```

## WAL
``` sql
-- кто (Список менеджеров ресурсов) пишет в WAL
# /usr/lib/postgresql/14/bin/pg_waldump -r list
SELECT * FROM pg_ls_waldir()


-- текущий номер записи в WAL
SELECT pg_current_wal_insert_lsn();
SELECT pg_walfile_name('0/BBFF128'); -- в каком файле сейчас

SELECT pg_current_wal_lsn(); -- номер записи журанала которая уже на диске 

--  файлы
# ls PGDATA/pg_wal

SHOW wal_buffers; -- буфер, 1/32 shared_buffers

CREATE EXTENSION pageinspect;

-- посмотреть WAL
# usr/lib/postgresql/14/bin/pg_waldump -p /var/lib/postgresql/14/main/pg_wal -s 2/44E2C0D0 -e 2/44E2C198
```

## CHECKPOINT

``` sql
-- файл содержит статус сервера, в т.ч. числе инфу по котрольным точкам
# /usr/lib/postgresql/14/bin/pg_controldata -D /var/lib/postgresql/14/main


checkpoint_timeout = 5min -- лучше 30 минут
checkpoint_completion_target = 0.9 -- сколько работать, а сколько ждать в процентах между CP
max_wal_size = 1GB  -- при достижении вызвать CP
min_wal_size = 100MB -- не снижаемый остаток, чтоб не частить

-- дополнительно работает bgwriter
bgwriter_delay = 200ms              -- засыпаем на это время
bgwriter_lru_maxpages = 100         -- страниц за цикл 
bgwriter_lru_multiplier = 2.0       -- если хорошо пошло, то в след.итерацию будет в два раза больше чем сейчас, для лавинной нагрузки

-- 
checkpoint_warning = 30s            -- выводит в журнал варнинги если CP сыпятся часто 
log_checkpoints = off               -- выводить о каждой CP инфу

-- статистика  
SELECT * FROM pg_stat_bgwriter \gx

* checkpoints_timed — контрольные точки по расписанию (checkpoint_timeout);
* checkpoints_req — контрольные точки по требованию (max_wal_size);
* buffers_checkpoint — страницы, сброшенные при контрольных точках;
* buffers_backend — страницы, сброшенные обслуживающими процессами;
* buffers_clean — страницы, сброшенные процессом фоновой записи.

-- уровни журнала 
wal_level = replica


-- контрольные суммы 
data_checksums = off
ignore_checksum_failure = off

-- управлять
# /usr/lib/postgresql/14/bin/pg_checksums



-- Синхронный режим
synchronous_commit = on      -- режим 
commit_delay = 0             -- пауза 
commit_siblings = 5          -- при накоплении не менее чем 

-- Асинхронный, процесс wal writer,
wal_writer_delay = 200ms        -- частота
wal_writer_flush_after = 1MB    -- размер накопления


```

