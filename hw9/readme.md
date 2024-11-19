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


pg_dump -Fc -Z -C -n hw9 -f /backup/archbkp.gz 

pg_dump testbkp -Fc -Z -C  -n xyz -f /backup/bkp.gz -U postgres -W

Используя утилиту pg_restore восстановим в новую БД только вторую таблицу!
