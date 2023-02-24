# ДЗ 03. Установка и настройка PostgreSQL


## сделаем виртуалку в GCE и поднимем там 14 postgresql и сделаем табличку 
```
# в терминале GOOGLE CLOUD SHELL делаем VM
gcloud compute instances create vm-postgresql \
    --project postgresql-linux-2023 \
    --zone=europe-north1-a \
    --machine-type=e2-small \
    --create-disk=auto-delete=yes,boot=yes,image=projects/ubuntu-os-cloud/global/images/ubuntu-2004-focal-v20230213,mode=rw,size=10 

# заходим в новую VM
PS C:\p\otus.postgresql.code> ssh -i gcp ae@34.88.169.125

# ставим postgresql 14-sq
sudo apt update && sudo apt upgrade -y && sudo sh -c 'echo "deb http://apt.postgresql.org/pub/repos/apt $(lsb_release -cs)-pgdg main" > /etc/apt/sources.list.d/pgdg.list' && wget --quiet -O - https://www.postgresql.org/media/keys/ACCC4CF8.asc | sudo apt-key add - && sudo apt-get update && sudo apt-get -y install postgresql-14

# создадим табличку в БД postgres
root@vm-postgresql:/home/ae# sudo -u postgres psql -p 5432
psql (14.7 (Ubuntu 14.7-1.pgdg20.04+1))
Type "help" for help.

postgres=# create table test (i int);
CREATE TABLE
postgres=# insert into test values (2023);
INSERT 0 1
postgres=# select * from test;
  i
------
 2023
(1 row)
# всё нормально, табличка есть 
```

## сделаем диск новый в GCE и подключим его VM
```
# в GOOGLE CLOUD SHELL делаем доп.диск 
gcloud compute disks create disk-1 --project=postgresql-linux-2023 --size=10GB --zone=europe-north1-a

# примаунтим его к VM 
gcloud compute instances attach-disk vm-postgresql --disk disk-1 --zone=europe-north1-a

# в VM форматим и моунтим его в /mnt/data
mkdir -p /mnt/data
sgdisk --new 1::0  /dev/sdb
mkfs.ext4 /dev/sdb1
lsblk --fs
vi /etc/fstab

UUID=6495ecf5-ed61-4dba-939a-95360a60612c /mnt/data ext4 defaults 0 2

mount -a
chown -R postgres:postgres /mnt/data/
```

## перенесем данные postgresql на этот новый диск
```
# стопаем postgresql
pg_ctlcluster 14 main stop

# переносим каталог с данными 
mv /var/lib/postgresql/14 /mnt/data

# стартуем
pg_ctlcluster 14 main start

# ошибка ожидаемо 
Error: /var/lib/postgresql/14/main is not accessible or does not exist

# правим конфиг 
vi /etc/postgresql/14/main/postgresql.conf
data_directory = '/mnt/data/14/main'

# стартуем
sudo -u postgres pg_ctlcluster 14 main start

# всё ок
root@vm-prosgresql:/mnt/data# systemctl status postgresql
● postgresql.service - PostgreSQL RDBMS
     Loaded: loaded (/lib/systemd/system/postgresql.service; enabled; vendor preset: enabled)
     Active: active (exited) since Fri 2023-02-24 12:09:04 UTC; 31min ago
   Main PID: 860 (code=exited, status=0/SUCCESS)
      Tasks: 0 (limit: 2356)
     Memory: 0B
     CGroup: /system.slice/postgresql.service

Feb 24 12:09:04 vm-prosgresql systemd[1]: Starting PostgreSQL RDBMS...
Feb 24 12:09:04 vm-prosgresql systemd[1]: Finished PostgreSQL RDBMS.

# проверям - есть табличка
sudo -u postgres psql -p 5432
postgres=# select * from test;
 c1
----
 1
(1 row)
``` 

## Задание под *: делаем еще одну VM, а старую стопаем и диск ее забираем
```
# делаем в google cloud еще одну виртуалку 
gcloud compute instances create vm-postgresql-2 \
    --project postgresql-linux-2023 \
    --zone=europe-north1-a \
    --machine-type=e2-small \
    --create-disk=auto-delete=yes,boot=yes,image=projects/ubuntu-os-cloud/global/images/ubuntu-2004-focal-v20230213,mode=rw,size=10 

# стопаем первую VM и забираем ее диск 
gcloud compute instances stop vm-postgresql --zone=europe-north1-a
gcloud compute instances detach-disk vm-prosgresql --disk disk-1 --zone=europe-north1-a

# подключаем диск к новой VM
gcloud compute instances attach-disk vm-postgresql-2 --disk disk-1

# снова ставим 14-ый postgresql
sudo apt update && sudo apt upgrade -y && sudo sh -c 'echo "deb http://apt.postgresql.org/pub/repos/apt $(lsb_release -cs)-pgdg main" > /etc/apt/sources.list.d/pgdg.list' && wget --quiet -O - https://www.postgresql.org/media/keys/ACCC4CF8.asc | sudo apt-key add - && sudo apt-get update && sudo apt-get -y install postgresql-14

root@vm-postgresql-2:/home/ae# pg_lsclusters
Ver Cluster Port Status Owner    Data directory              Log file
14  main    5432 online postgres /var/lib/postgresql/14/main /var/log/postgresql/postgresql-14-main.log

root@vm-postgresql-2:/home/ae# pg_ctlcluster 14 main stop

# правим конфиг 
vi /etc/postgresql/14/main/postgresql.conf
data_directory = '/mnt/data/14/main'

root@vm-postgresql-2:/home/ae# pg_ctlcluster 14 main start
root@vm-postgresql-2:/mnt/data/14/main# pg_lsclusters
Ver Cluster Port Status Owner    Data directory    Log file
14  main    5432 online postgres /mnt/data/14/main /var/log/postgresql/postgresql-14-main.log

# проверяем что там с табличкой
root@vm-postgresql-2:/mnt/data/14/main# sudo -u postgres psql -p 5432
postgres=# select * from test;
 c1
----
 1
(1 row)
# всё гуд
```

