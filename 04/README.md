## ДЗ

```
# 1
# ставим postgresql 14-ый
sudo apt update && sudo apt upgrade -y && sudo sh -c 'echo "deb http://apt.postgresql.org/pub/repos/apt $(lsb_release -cs)-pgdg main" > /etc/apt/sources.list.d/pgdg.list' && wget --quiet -O - https://www.postgresql.org/media/keys/ACCC4CF8.asc | sudo apt-key add - && sudo apt-get update && sudo apt-get -y install postgresql-14

# 2-7
# делаем БД, схему в ней, табличку и вставить туда чего-нибудть 
postgres=# create database testdb;
CREATE DATABASE
postgres=# \c testdb
You are now connected to database "testdb" as user "postgres".
testdb=# create schema testnm;
CREATE SCHEMA
testdb=# create table t1 (i int);
CREATE TABLE
testdb=# insert into t1 values (2023);
INSERT 0 1

# 8-13
# делаем роль,  
testdb=# create role readonly;
CREATE ROLE
testdb=# grant connect on database testdb TO readonly;
GRANT
testdb=# grant usage on schema testnm to readonly;
GRANT
testdb=# grant select on all tables in schema testnm TO readonly;    
GRANT
testdb=# create role testread with login password 'test123';
CREATE ROLE
testdb=# grant readonly to testread;
GRANT ROLE
testdb=# \du
                                    List of roles
 Role name |                         Attributes                         | Member of
-----------+------------------------------------------------------------+------------
 postgres  | Superuser, Create role, Create DB, Replication, Bypass RLS | {}
 readonly  | Cannot login                                               | {}
 testread  |                                                            | {readonly}

# 14
# подключимся как testread
testdb=# \c testdb testread
connection to server on socket "/var/run/postgresql/.s.PGSQL.5432" failed: FATAL:  Peer authentication failed for user "testread"
Previous connection kept

# подключаемся не через peer, а по сети localhost (чтоб пароль спросило), иначе надо в файле pg_hba.conf менять peer на md5 для локального входа
root@pg-05:/home/ae1# sudo -u postgres psql -U testread -h localhost -p 5432 -d testdb
Password for user testread:
psql (14.7 (Ubuntu 14.7-1.pgdg20.04+1))
SSL connection (protocol: TLSv1.3, cipher: TLS_AES_256_GCM_SHA384, bits: 256, compression: off)
Type "help" for help.
testdb=>

# 15-21
# проверим 
testdb=> select * from t1;
ERROR:  permission denied for table t1
testdb=> \dt
        List of relations
 Schema | Name | Type  |  Owner
--------+------+-------+----------
 public | t1   | table | postgres
(1 row)
# не сработало, т.к. табличка в public создалась, т.к. в момент ее создания схемы testuser небыло
# посмотреть какой порядок поиска в переменной: 
select search_path

# 22-25
# заходим снова под postgres, дропаем и делаем заново табличку в нужной схеме
root@pg-05:/home/ae1# sudo -u postgres psql
psql (14.7 (Ubuntu 14.7-1.pgdg20.04+1))
Type "help" for help.
postgres=#
testdb=# drop table t1;
DROP TABLE
testdb=# create table testnm.t1 (i int);
CREATE TABLE
testdb=# insert into testnm.t1 values (2023);
INSERT 0 1

# 26-33
# проверяем 
root@pg-05:/home/ae1# sudo -u postgres psql -U testread -h localhost -p 5432 -d testdb
Password for user testread:
psql (14.7 (Ubuntu 14.7-1.pgdg20.04+1))
SSL connection (protocol: TLSv1.3, cipher: TLS_AES_256_GCM_SHA384, bits: 256, compression: off)
Type "help" for help.

testdb=> select * from  testnm.t1;
ERROR:  permission denied for table t1
testdb=> 
# неа, табличка создана после гранта прав на схему, надо было либо раньше делать дефолтовые разрешения
alter default privileges in schema testnm grant select on tables to readonly
и пересоздать табличку
либо снова 
grant select on all tables in schema testnm TO readonly.
Сделаем и то и то: 
root@pg-05:/home/ae1# sudo -u postgres psql
psql (14.7 (Ubuntu 14.7-1.pgdg20.04+1))
Type "help" for help.

postgres=# \c testdb
You are now connected to database "testdb" as user "postgres".
testdb=# alter default privileges in schema testnm grant select on tables to readonly;
ALTER DEFAULT PRIVILEGES
testdb=# grant select on all tables in schema testnm TO readonly;
GRANT
testdb=# \q
root@pg-05:/home/ae1# sudo -u postgres psql -U testread -h localhost -p 5432 -d testdb
Password for user testread:
psql (14.7 (Ubuntu 14.7-1.pgdg20.04+1))
SSL connection (protocol: TLSv1.3, cipher: TLS_AES_256_GCM_SHA384, bits: 256, compression: off)
Type "help" for help.

testdb=> select * from testnm.t1;
  i
------
 2023
(1 row)

testdb=>


```

## на память
### 1. запрет создания объектов в схеме public
т.к. лучше убрать возможность что-либо создавать в схеме public кому попало, т.к. search_path указывает в первую очередь на схему public. А схема public создается в каждой базе данных по умолчанию. И grant на все действия в этой схеме дается роли public. А роль public добавляется всем новым пользователям. Соответсвенно каждый пользователь может по умолчанию создавать объекты в схеме public любой базы данных, ес-но если у него есть право на подключение к этой базе данных. Чтобы раз и навсегда забыть про роль public - а в продакшн базе данных про нее лучше забыть - выполните следующие действия 
```
# 34-39
# любой кто имеет доступ к public может насоздовать там объектов
testdb=> create table t2 (i int);
CREATE TABLE
# заберем права
\c testdb postgres; 
revoke CREATE on SCHEMA public FROM public; 
revoke all on DATABASE testdb FROM public; 
\c testdb testread; 

# пробуем создать табличку
testdb=> create table t3 (i int);
ERROR:  permission denied for schema public
LINE 1: create table t3 (i int);

# всё отлично, так и надо!

```







