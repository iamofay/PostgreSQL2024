### Бэкапы

Цель:
- применить логический бэкап. Восстановиться из бэкапа.

#### Используем ВМ с ubuntu и установленным pgsql 15, которая ранее использовалась для других ДЗ

```
daa@daa-pgsql:~$ sudo -u postgres pg_lsclusters
Ver Cluster Port Status Owner    Data directory              Log file
15  main    5432 online postgres /var/lib/postgresql/15/main /var/log/postgresql/postgresql-15-main.log
```

#### Подключимся к БД

```
daa@daa-pgsql:~$ sudo -u postgres psql
could not change directory to "/home/daa": Permission denied
psql (15.9 (Ubuntu 15.9-1.pgdg24.04+1))
Type "help" for help.

postgres=# 
```

#### Создадим БД, схему и в ней таблицу

```
postgres=# create database testbkp;
CREATE DATABASE
postgres=# \c testbkp
You are now connected to database "testbkp" as user "postgres".
testbkp=# create schema hw9;
CREATE SCHEMA
testbkp=#  CREATE TABLE hw9.rnd  (
    id SERIAL PRIMARY KEY,
    text TEXT
);
CREATE TABLE
```

#### Заполним таблицук сгенерированными 100 записями

```
testbkp=#INSERT INTO hw9.rnd
SELECT id, MD5(random()::TEXT)::TEXT
FROM generate_series(1, 100) AS id;
INSERT 0 100
testbkp=# select * from hw9.rnd limit 5;
 id |               text
----+----------------------------------
  1 | d73a4ab6263d0bb7fb9e45af71e0ed44
  2 | 5d6b86798a46a2fa0d264eec8a3dc8ba
  3 | b7fda20bb87cfdb9a654d546c90498dd
  4 | 106274cd7f2e1c3f8606457eb6e8e221
  5 | 148fd98497a7208860b545f4bab5d382
(5 rows)
```

#### Под линукс пользователем Postgres создадим каталог для бэкапов

```
daa@daa-pgsql:~$ sudo mkdir /backup
[sudo] password for daa:
daa@daa-pgsql:~$ sudo chown -R postgres: /backup
```

#### Сделаем логический бэкап используя утилиту COPY

```
testbkp=# copy hw9.rnd TO '/backup/rnd';
COPY 100
```


#### Восстановим в 2 таблицу данные из бэкапа.

```
testbkp=# CREATE TABLE hw9.rnd2  (
    id SERIAL PRIMARY KEY,
    text TEXT
);
CREATE TABLE

testbkp=# copy hw9.rnd2 from '/backup/rnd';
COPY 100
```


#### Используя утилиту pg_dump создадим бэкап в кастомном сжатом формате двух таблиц

1. Создадим пользователя и дадим ему прав на создание бэкапа

```
testbkp=# CREATE USER bkptest WITH PASSWORD 'password';
CREATE ROLE
testbkp=# GRANT ALL PRIVILEGES ON DATABASE testbkp TO bkptest;
GRANT
testbkp=# GRANT ALL PRIVILEGES ON SCHEMA hw9 TO bkptest;
GRANT
testbkp=# GRANT ALL PRIVILEGES ON ALL TABLES IN SCHEMA hw9 TO bkptest;
GRANT
testbkp=# GRANT ALL ON ALL SEQUENCES IN SCHEMA hw9 TO bkptest;
GRANT
```

2. Дадим прав на подключение в pg_hba.conf

![image](https://github.com/user-attachments/assets/4ae9d8c7-c06a-408f-a7d8-ae9aa296e41e)

3. Создадим бэкап

```
daa@daa-pgsql:sudo pg_dump testbkp -h localhost -Fc -Z5 -C -n hw9 -f /backup/bkp.gz -U bkptest -W
```

4. Посмотрим на содержание

```
daa@daa-pgsql:/etc/postgresql/15/main$ pg_restore -l /backup/bkp.gz
;
; Archive created at 2024-11-19 17:21:22 MSK
;     dbname: testbkp
;     TOC Entries: 25
;     Compression: 5
;     Dump Version: 1.14-0
;     Format: CUSTOM
;     Integer: 4 bytes
;     Offset: 8 bytes
;     Dumped from database version: 15.9 (Ubuntu 15.9-1.pgdg24.04+1)
;     Dumped by pg_dump version: 15.9 (Ubuntu 15.9-1.pgdg24.04+1)
;
;
; Selected TOC Entries:
;
6; 2615 16400 SCHEMA - hw9 postgres
3444; 0 0 ACL - SCHEMA hw9 postgres
216; 1259 16402 TABLE hw9 rnd postgres
3445; 0 0 ACL hw9 TABLE rnd postgres
218; 1259 16412 TABLE hw9 rnd2 postgres
3446; 0 0 ACL hw9 TABLE rnd2 postgres
217; 1259 16411 SEQUENCE hw9 rnd2_id_seq postgres
3447; 0 0 SEQUENCE OWNED BY hw9 rnd2_id_seq postgres
3448; 0 0 ACL hw9 SEQUENCE rnd2_id_seq postgres
215; 1259 16401 SEQUENCE hw9 rnd_id_seq postgres
3449; 0 0 SEQUENCE OWNED BY hw9 rnd_id_seq postgres
3450; 0 0 ACL hw9 SEQUENCE rnd_id_seq postgres
3285; 2604 16405 DEFAULT hw9 rnd id postgres
3286; 2604 16415 DEFAULT hw9 rnd2 id postgres
3434; 0 16402 TABLE DATA hw9 rnd postgres
3436; 0 16412 TABLE DATA hw9 rnd2 postgres
3451; 0 0 SEQUENCE SET hw9 rnd2_id_seq postgres
3452; 0 0 SEQUENCE SET hw9 rnd_id_seq postgres
3290; 2606 16419 CONSTRAINT hw9 rnd2 rnd2_pkey postgres
3288; 2606 16409 CONSTRAINT hw9 rnd rnd_pkey postgres
```

5. Сохраним в отдельный файл контент для восстановления связанный с таблицей bkp2

![image](https://github.com/user-attachments/assets/b6ee6bc9-c165-4ce0-b5b1-a19aff021c74)

6. Создадим новую БД, и дадим прав для уз bkptest

```
postgres=# CREATE DATABASE testbkp2;
CREATE DATABASE
postgres=# \c testbkp2
You are now connected to database "testbkp2" as user "postgres".
testbkp2=# GRANT ALL PRIVILEGES ON DATABASE testbkp2 TO bkptest;
GRANT
testbkp2=# ALTER DEFAULT PRIVILEGES IN SCHEMA public GRANT ALL ON TABLES TO bkptest;
ALTER DEFAULT PRIVILEGES
```

7. Восстановим таблицу из бэкапа в новой БД

```
daa@daa-pgsql:/backup$ pg_restore -h localhost -L /backup/bkp.list -d testbkp2 -U bkptest -W /backup/bkp.gz
```

8. Проверим результат в новой БД

```
testbkp2=# select * from hw9.rnd2 limit 5;
id |               text
----+----------------------------------
  1 | d73a4ab6263d0bb7fb9e45af71e0ed44
  2 | 5d6b86798a46a2fa0d264eec8a3dc8ba
  3 | b7fda20bb87cfdb9a654d546c90498dd
  4 | 106274cd7f2e1c3f8606457eb6e8e221
  5 | 148fd98497a7208860b545f4bab5d382
(5 rows)

testbkp2=# \dt hw9.*
        List of relations
 Schema | Name | Type  |  Owner
--------+------+-------+---------
 hw9    | rnd2 | table | bkptest
(1 row)
```
