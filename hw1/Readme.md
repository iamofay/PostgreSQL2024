### Работа с уровнями изоляции транзакции в PostgreSQL

Цель:
-научиться работать в Яндекс Облаке
-научиться управлять уровнем изолции транзации в PostgreSQL и понимать особенность работы уровней read commited и repeatable read



Конфигурация для запуска postgres: [docker-compose.yaml](./docker-compose.yaml "docker compose")

## Запуск в контейнере
```
podman-compose up -d
podman-compose down
```

### Подключение к базе данных:
```
podman run --rm -it --network frontend --env-file .env postgres:16.1 bash -c 'export PGPASSWORD=$PG_PASS; psql -h db -U $PG_USER $PG_DB'
```

## run postgres on ubuntu by qemu

Скрипт: [qemu_ubuntu.sh](./qemu_ubuntu.sh "qemu run scritp" )

### Свежая ubuntu
```
wget "https://cloud-images.ubuntu.com/noble/current/noble-server-cloudimg-amd64.img" -O noble-server-cloudimg-amd64.img
```

### Сделаем снапшот с базового образа
```
qemu-img create -f qcow2 -b noble-server-cloudimg-amd64.img -F qcow2 noble.img
```

### Описание форматра cloud-init:
https://cloudinit.readthedocs.io/en/latest/reference/examples.html

### Описание запуска ubuntu c cloud-init qemu и вебсервера python
https://cloudinit.readthedocs.io/en/latest/tutorial/qemu.html

### Запустим веб сервер

```
python3 -m http.server --directory cloud/12
```

### Запустим вирутальную машину

```
./qemu_ubuntu.sh 
```

### Подключемся к vm

```sh
ssh -l ansible localhost -p 10023 -o "StrictHostKeyChecking=no"
```
Или добавить в ~/.ssh/config 
```
Host pg0
    HostName localhost
    Port 10023
    StrictHostKeyChecking no
    User ansible
```
```sh 
ssh pg0
```

Подключимся к уже созданой VM
### pg_lsclusters

```
$ pg_lsclusters 
Ver Cluster Port Status Owner    Data directory              Log file
12  main    5432 online postgres /var/lib/postgresql/12/main /var/log/postgresql/postgresql-12-main.log
```

### hba default host encryption scram-sha-256
$podman run --rm -it mirror.gcr.io/postgres:15 cat /usr/share/postgresql/postgresql.conf.sample | grep password_encryption

```
#password_encryption = scram-sha-256	# scram-sha-256 or md5
```

$podman run --rm -it mirror.gcr.io/postgres:13 cat /usr/share/postgresql/postgresql.conf.sample | grep password_encryption

```
#password_encryption = md5		# md5 or scram-sha-256
```

## Isolation
Воспользуемся VM ./qemu_ubuntu.sh
```sh
sudo apt-get install postgresql-15 postgresql-14
systemctl stop postgresql
```

```sh
$sudo -u postgres pg_createcluster 14 master
Creating new PostgreSQL cluster 14/master ...
/usr/lib/postgresql/14/bin/initdb -D /var/lib/postgresql/14/master --auth-local peer --auth-host scram-sha-256 --no-instructions
The files belonging to this database system will be owned by user "postgres".
This user must also own the server process.

The database cluster will be initialized with locale "C.UTF-8".
The default database encoding has accordingly been set to "UTF8".
The default text search configuration will be set to "english".

Data page checksums are disabled.

fixing permissions on existing directory /var/lib/postgresql/14/master ... ok
creating subdirectories ... ok
selecting dynamic shared memory implementation ... posix
selecting default max_connections ... 100
selecting default shared_buffers ... 128MB
selecting default time zone ... Etc/UTC
creating configuration files ... ok
running bootstrap script ... ok
performing post-bootstrap initialization ... ok
syncing data to disk ... ok
Warning: systemd does not know about the new cluster yet. Operations like "service postgresql start" will not handle it. To fix, run:
  sudo systemctl daemon-reload
Ver Cluster Port Status Owner    Data directory                Log file
14  master  5435 down   postgres /var/lib/postgresql/14/master /var/log/postgresql/postgresql-14-master.log
```

### pg_lsclusters
```sh
Ver Cluster Port Status Owner    Data directory                Log file
12  main    5432 down   postgres /var/lib/postgresql/12/main   /var/log/postgresql/postgresql-12-main.log
14  main    5434 down   postgres /var/lib/postgresql/14/main   /var/log/postgresql/postgresql-14-main.log
14  master  5435 down   postgres /var/lib/postgresql/14/master /var/log/postgresql/postgresql-14-master.log
15  main    5433 down   postgres /var/lib/postgresql/15/main   /var/log/postgresql/postgresql-15-main.log
```

### sudo pg_ctlcluster 14 master start

$sudo pg_ctlcluster 14 master start
$pg_lsclusters

```sh
Ver Cluster Port Status Owner    Data directory                Log file
12  main    5432 down   postgres /var/lib/postgresql/12/main   /var/log/postgresql/postgresql-12-main.log
14  main    5434 down   postgres /var/lib/postgresql/14/main   /var/log/postgresql/postgresql-14-main.log
14  master  5435 online postgres /var/lib/postgresql/14/master /var/log/postgresql/postgresql-14-master.log
15  main    5433 down   postgres /var/lib/postgresql/15/main   /var/log/postgresql/postgresql-15-main.log
```

### sudo -u postgres psql -p 5435
```sh
psql (15.7 (Ubuntu 15.7-1.pgdg24.04+1), server 14.12 (Ubuntu 14.12-1.pgdg24.04+1))
Type "help" for help.

postgres=# 
```

### создадим табличку для тестов

```sql
postgres=# CREATE DATABASE iso;
CREATE DATABASE
postgres=# \c iso 
psql (15.7 (Ubuntu 15.7-1.pgdg24.04+1), server 14.12 (Ubuntu 14.12-1.pgdg24.04+1))
You are now connected to database "iso" as user "postgres".
iso=# \l
                                             List of databases
   Name    |  Owner   | Encoding | Collate |  Ctype  | ICU Locale | Locale Provider |   Access privileges   
-----------+----------+----------+---------+---------+------------+-----------------+-----------------------
 iso       | postgres | UTF8     | C.UTF-8 | C.UTF-8 |            | libc            | 
 postgres  | postgres | UTF8     | C.UTF-8 | C.UTF-8 |            | libc            | 
 template0 | postgres | UTF8     | C.UTF-8 | C.UTF-8 |            | libc            | =c/postgres          +
           |          |          |         |         |            |                 | postgres=CTc/postgres
 template1 | postgres | UTF8     | C.UTF-8 | C.UTF-8 |            | libc            | =c/postgres          +
           |          |          |         |         |            |                 | postgres=CTc/postgres
(4 rows)

iso=# SELECT current_database();
CREATE TABLE test (i serial, amount int);
INSERT INTO test(amount) VALUES (100);
INSERT INTO test(amount) VALUES (500);
SELECT * FROM test;
 current_database 
------------------
 iso
(1 row)

CREATE TABLE
INSERT 0 1
INSERT 0 1
 i | amount 
---+--------
 1 |    100
 2 |    500
(2 rows)
iso=# \echo :AUTOCOMMIT
on
iso=# \set AUTOCOMMIT OFF
iso=# show transaction isolation level;
 transaction_isolation 
-----------------------
 read committed
(1 row)

iso=*# set transaction isolation level read committed;
SET
iso=*# set transaction isolation level repeatable read;
SET
iso=*# set transaction isolation level serializable;
SET
iso=*# SELECT txid_current();
 txid_current 
--------------
          738
(1 row)

iso=*# \set AUTOCOMMIT ON
iso=*# SELECT txid_current();
 txid_current 
--------------
          738
(1 row)

iso=*# SELECT * FROM test;
 i | amount 
---+--------
 1 |    100
 2 |    500
(2 rows)

iso=*# commit;
COMMIT

iso=# \x on \pset pager 0
Expanded display is on.
Pager usage is off.

iso=# SELECT * FROM pg_stat_activity;
-[ RECORD 1 ]----+--------------------------------
datid            | 
datname          | 
pid              | 4917
leader_pid       | 
usesysid         | 
usename          | 
application_name | 
client_addr      | 
client_hostname  | 
client_port      | 
backend_start    | 2024-05-30 07:11:57.124861+00
xact_start       | 
query_start      | 
state_change     | 
wait_event_type  | Activity
wait_event       | AutoVacuumMain
state            | 
backend_xid      | 
backend_xmin     | 
query_id         | 
query            | 
backend_type     | autovacuum launcher
-[ RECORD 2 ]----+--------------------------------
datid            | 
datname          | 
pid              | 4919
leader_pid       | 
usesysid         | 10
usename          | postgres
application_name | 
client_addr      | 
client_hostname  | 
client_port      | 
backend_start    | 2024-05-30 07:11:57.125654+00
xact_start       | 
query_start      | 
state_change     | 
wait_event_type  | Activity
wait_event       | LogicalLauncherMain
state            | 
backend_xid      | 
backend_xmin     | 
query_id         | 
query            | 
backend_type     | logical replication launcher
-[ RECORD 3 ]----+--------------------------------
datid            | 16384
datname          | iso
pid              | 5216
leader_pid       | 
usesysid         | 10
usename          | postgres
application_name | psql
client_addr      | 
client_hostname  | 
client_port      | -1
backend_start    | 2024-05-30 07:44:06.107568+00
xact_start       | 2024-05-30 07:53:31.512879+00
query_start      | 2024-05-30 07:53:31.512879+00
state_change     | 2024-05-30 07:53:31.512883+00
wait_event_type  | 
wait_event       | 
state            | active
backend_xid      | 
backend_xmin     | 740
query_id         | 
query            | SELECT * FROM pg_stat_activity;
backend_type     | client backend
-[ RECORD 4 ]----+--------------------------------
datid            | 
datname          | 
pid              | 4915
leader_pid       | 
usesysid         | 
usename          | 
application_name | 
client_addr      | 
client_hostname  | 
client_port      | 
backend_start    | 2024-05-30 07:11:57.124146+00
xact_start       | 
query_start      | 
state_change     | 
wait_event_type  | Activity
wait_event       | BgWriterHibernate
state            | 
backend_xid      | 
backend_xmin     | 
query_id         | 
query            | 
backend_type     | background writer
-[ RECORD 5 ]----+--------------------------------
datid            | 
datname          | 
pid              | 4914
leader_pid       | 
usesysid         | 
usename          | 
application_name | 
client_addr      | 
client_hostname  | 
client_port      | 
backend_start    | 2024-05-30 07:11:57.123495+00
xact_start       | 
query_start      | 
state_change     | 
wait_event_type  | Activity
wait_event       | CheckpointerMain
state            | 
backend_xid      | 
backend_xmin     | 
query_id         | 
query            | 
backend_type     | checkpointer
-[ RECORD 6 ]----+--------------------------------
datid            | 
datname          | 
pid              | 4916
leader_pid       | 
usesysid         | 
usename          | 
application_name | 
client_addr      | 
client_hostname  | 
client_port      | 
backend_start    | 2024-05-30 07:11:57.124518+00
xact_start       | 
query_start      | 
state_change     | 
wait_event_type  | Activity
wait_event       | WalWriterMain
state            | 
backend_xid      | 
backend_xmin     | 
query_id         | 
query            | 
backend_type     | walwriter
```

### -- test TRANSACTION ISOLATION LEVEL READ COMMITTED;

> console 1
```sql
\c iso 
psql (15.7 (Ubuntu 15.7-1.pgdg24.04+1), server 14.12 (Ubuntu 14.12-1.pgdg24.04+1))
You are now connected to database "iso" as user "postgres".
iso=# BEGIN;
BEGIN
iso=*# SELECT * FROM test;
 i | amount 
---+--------
 1 |    100
 2 |    500
(2 rows)
```

> console 2
```sql
\c iso 
psql (15.7 (Ubuntu 15.7-1.pgdg24.04+1), server 14.12 (Ubuntu 14.12-1.pgdg24.04+1))
You are now connected to database "iso" as user "postgres".
iso=# BEGIN;
UPDATE test set amount = 555 WHERE i = 1;
COMMIT;
BEGIN
UPDATE 1
COMMIT
```

Видим изменения вызваные транзакцией из 2й консоли
> console 1
```sql
SELECT * FROM test; -- different values
COMMIT;
 i | amount 
---+--------
 2 |    500
 1 |    555
(2 rows)

COMMIT

```

### -- TRANSACTION ISOLATION LEVEL REPEATABLE READ;

> console 1
```sql
 BEGIN TRANSACTION ISOLATION LEVEL REPEATABLE READ;
SELECT * FROM test;
BEGIN
 i | amount 
---+--------
 2 |    500
 1 |    555
(2 rows)
```

> console 2
```sql
BEGIN TRANSACTION ISOLATION LEVEL REPEATABLE READ;
INSERT INTO test VALUES (777);
COMMIT;
BEGIN
INSERT 0 1
COMMIT
```

Теперь при степени изоляции REPEATABLE READ мы не видим изменения транзакции из второй косоли;
> console 1

```sql
iso=*# SELECT * FROM test;
 i | amount 
---+--------
 2 |    500
 1 |    555
(2 rows)
```

### -- TRANSACTION ISOLATION LEVEL SERIALIZABLE;
```sql
DROP TABLE IF EXISTS testS;
CREATE TABLE testS (i int, amount int);
INSERT INTO TESTS VALUES (1,10), (1,20), (2,100), (2,200); 
DROP TABLE
CREATE TABLE
INSERT 0 4
```

```sql
-- 1 console

BEGIN TRANSACTION ISOLATION LEVEL SERIALIZABLE;
SELECT sum(amount) FROM testS WHERE i = 1;
INSERT INTO testS VALUES (2,30);
BEGIN
 sum 
-----
  30
(1 row)

INSERT 0 1
```
```sql
-- 2 consol
BEGIN TRANSACTION ISOLATION LEVEL SERIALIZABLE;
SELECT sum(amount) FROM testS WHERE i = 2;
INSERT INTO testS VALUES (1,300);
BEGIN
 sum 
-----
 300
(1 row)

INSERT 0 1
```

```sql
-- 1 console
COMMIT;
COMMIT
```

Получаем ERROR не могут быть сериализованны
```sql
-- 2 consol
COMMIT;
ERROR:  could not serialize access due to read/write dependencies among transactions
DETAIL:  Reason code: Canceled on identification as a pivot, during commit attempt.
HINT:  The transaction might succeed if retried.
```

Если повторить то же самое с другим уровнем изоляции REPEATABLE READ
```sql
-- 1 console
iso=# BEGIN TRANSACTION ISOLATION LEVEL REPEATABLE READ;
SELECT sum(amount) FROM testS WHERE i = 1;
INSERT INTO testS VALUES (2,30);
BEGIN
 sum 
-----
  30
(1 row)
```
```sql
-- 2 console

 BEGIN TRANSACTION ISOLATION LEVEL REPEATABLE READ;
SELECT sum(amount) FROM testS WHERE i = 2;
INSERT INTO testS VALUES (1,300);
BEGIN
 sum 
-----
 330
(1 row)
```
```sql
-- 1 console
INSERT 0 1
iso=*# COMMIT;
COMMIT
```

```sql
-- 2 console
INSERT 0 1
iso=*# COMMIT;
COMMIT
```
Обе тразакции успешны

# Домашнее задание 01


  в первой сессии новую таблицу и наполнить ее данными create table persons(id serial, first_name text, second_name text); insert into persons(first_name, second_name) values('ivan', 'ivanov'); insert into persons(first_name, second_name) values('petr', 'petrov'); commit;

```sql
postgres=# \set AUTOCOMMIT OFF
postgres=# \echo :AUTOCOMMIT
OFF
postgres=# create table persons(id serial, first_name text, second_name text); insert into persons(first_name, second_name) values('ivan', 'ivanov'); insert into persons(first_name, second_name) values('petr', 'petrov'); commit;
CREATE TABLE
INSERT 0 1
INSERT 0 1
COMMIT
```

  посмотреть текущий уровень изоляции: show transaction isolation level
```sql
postgres=# show transaction isolation level
postgres-# ;
 transaction_isolation 
-----------------------
 read committed
(1 row)
```
  начать новую транзакцию в обоих сессиях с дефолтным (не меняя) уровнем изоляции
  в первой сессии добавить новую запись insert into persons(first_name, second_name) values('sergey', 'sergeev');
```sql
postgres=*# insert into persons(first_name, second_name) values('sergey', 'sergeev');
INSERT 0 1
``` 
  сделать select from persons во второй сессии
```sql
select * from persons;
 id | first_name | second_name 
----+------------+-------------
  1 | ivan       | ivanov
  2 | petr       | petrov
(2 rows)
```    
  видите ли вы новую запись и если да то почему?
```
Не видим потмоу что автокоммит отключен. и уровень изоляции по умолчанию read commited.
```
  завершить первую транзакцию - commit;
```sql
postgres=*# commit;
COMMIT
```    
  сделать select from persons во второй сессии
```sql
postgres=*# select * from persons 
postgres-*# ;
 id | first_name | second_name 
----+------------+-------------
  1 | ivan       | ivanov
  2 | petr       | petrov
  3 | sergey     | sergeev
(3 rows)
```
  видите ли вы новую запись и если да то почему?
```
Запись появилась потому что транзакция со вставкой завершена.
```    
  завершите транзакцию во второй сессии
```sql
postgres=*# commit;
COMMIT
```
  начать новые но уже repeatable read транзации - set transaction isolation level repeatable read;
  в первой сессии добавить новую запись insert into persons(first_name, second_name) values('sveta', 'svetova');
```sql
postgres=# set transaction isolation level repeatable read;
SET
postgres=*# insert into persons(first_name, second_name) values('sveta', 'svetova');
INSERT 0 1
```
  сделать select* from persons во второй сессии*
```sql
postgres=# set transaction isolation level repeatable read;
SET
postgres=*# select* from persons;
 id | first_name | second_name 
----+------------+-------------
  1 | ivan       | ivanov
  2 | petr       | petrov
  3 | sergey     | sergeev
(3 rows)
```
  видите ли вы новую запись и если да то почему?
```
Нет так как данные еше не добавлены первой транзакцией. И она еще не завершена. 
```
  завершить первую транзакцию - commit;
```sql
postgres=*# commit;
COMMIT
```
  сделать select from persons во второй сессии
```sql
postgres=*# select * from persons;
 id | first_name | second_name 
----+------------+-------------
  1 | ivan       | ivanov
  2 | petr       | petrov
  3 | sergey     | sergeev
(3 rows)
```    
  видите ли вы новую запись и если да то почему?
```
Нет не видим, так как теперь у нас не завершенная транзакция в первой сессии - с уровнем изоляции repeatable read. Данные от первой появились, но для транзакции во второй сессии они не видны.
```
  завершить вторую транзакцию
```
postgres=*# commit;
COMMIT
```    
  сделать select * from persons во второй сессии
```
postgres=# select * from persons ;
 id | first_name | second_name 
----+------------+-------------
  1 | ivan       | ivanov
  2 | petr       | petrov
  3 | sergey     | sergeev
  4 | sveta      | svetova
(4 rows)
```    
  видите ли вы новую запись и если да то почему? ДЗ сдаем в виде миниотчета в markdown в гите
```
Так как начась новая транзакция во второй сессии на данном горизонте событий мы видим все предыдущие завершенные транзакции.
```
------
