# ДЗ 01. Работа с уровнями изоляции транзакции в PostgreSQL

## Инфраструктура 
Всё осталось от домашки [№02](https://github.com/BerdnikovAE/otus.postgresql/tree/main/02). 
<br>Запускаем в одном окне (#1) докер клиент, а во втором (#2) подключаемся и запускаем еще один ```psql``` 

## Подготовка
### #1 - подготовительные работы 
```
otus=# \set AUTOCOMMIT OFF
otus=# create table persons(id serial, first_name text, second_name text); insert into persons(first_name, second_name) values('ivan', 'ivanov'); insert into persons(first_name, second_name) values('petr', 'petrov'); commit;
CREATE TABLE
INSERT 0 1
INSERT 0 1
COMMIT
otus=# show transaction isolation level;
 transaction_isolation
-----------------------
 read committed
(1 row)
```
### #2 - аналогично и тут 
```
otus=# \set AUTOCOMMIT OFF
otus=# show transaction isolation level;
 transaction_isolation
-----------------------
 read committed
(1 row)
```

----------------
## read committed

### #1 - вставляем строчку и не комитим
```
otus=*# insert into persons(first_name, second_name) values('sergey', 'sergeev');
INSERT 0 1
```
### #2 - нет её тут 
```
otus=*# select * from persons;
 id | first_name | second_name
----+------------+-------------
  1 | ivan       | ivanov
  2 | petr       | petrov
(2 rows)
```
### #1 - комитим
```
otus=*# commit;
COMMIT
```
### #2 - есть строчка  
```
otus=*# select * from persons;
 id | first_name | second_name
----+------------+-------------
  1 | ivan       | ivanov
  2 | petr       | petrov
  3 | sergey     | sergeev
(3 rows)
otus=*# commit;
```
Вывод: всё правильно - в режиме ```read committed``` (дефолтовый режи) мы видим "Неповторяемое чтение", т.е. видим в свой транзакции только закомиченные другие транзакции.

-----------------------
## repeatable read

### #1 - вставляем строчку и не комитим
```
otus=*# set transaction isolation level repeatable read;
SET
otus=*# insert into persons(first_name, second_name) values('sveta', 'svetova');
INSERT 0 1
otus=*#
```
### #2 - ожидаемо ее не видим
```
otus=*# set transaction isolation level repeatable read;
SET
otus=*# select * from persons;
 id | first_name | second_name
----+------------+-------------
  1 | ivan       | ivanov
  2 | petr       | petrov
  3 | sergey     | sergeev
(3 rows)
```
### #1 - комитим
```
otus=*# commit;
COMMIT
```
### #2 - всё равно нету Светы
```
otus=*# select * from persons;
 id | first_name | second_name
----+------------+-------------
  1 | ivan       | ivanov
  2 | petr       | petrov
  3 | sergey     | sergeev
(3 rows)
```
### #2 - комитим и Света появляется 
```
otus=*# commit;
COMMIT
otus=# select * from persons;
 id | first_name | second_name
----+------------+-------------
  1 | ivan       | ivanov
  2 | petr       | petrov
  3 | sergey     | sergeev
  8 | sveta      | svetova
(4 rows)
```
Вывод: всё правильно - в режиме repeatable read мы НЕ видим "Неповторяемое чтение", т.е. видим в свой транзакции данные которые были зафиксированы до начала транзакции.

