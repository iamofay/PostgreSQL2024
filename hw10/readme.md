### Репликация

Цель:
- реализовать свой миникластер на 3 ВМ.

#### Создадим 4 вм

Для реализации домашнего задания использовалась VMWare Worstation, была поднята ВМ с ОС Ubuntu 20.04.1 desaktop

![image](https://github.com/user-attachments/assets/d1e3159a-9cd0-4c72-a9cb-97665e19c022)

```
hw10-1 192.168.1.106
hw10-1 192.168.1.107
hw10-1 192.168.1.108
hw10-1 192.168.1.109
```

#### Подготовим все ВМ

1. Разрешим подключения к БД

```
daa@daa-pgsql:~$ echo "listen_addresses = '*'" | sudo tee -a /etc/postgresql/15/main/postgresql.conf
[sudo] password for daa:
listen_addresses = '*'
```

2. Определим уровень логирования, уровень wal_level должен быть replica или выше

```
daa@daa-pgsql:~$ echo "wal_level = 'logical'" | sudo tee -a /etc/postgresql/15/main/postgresql.conf
wal_level = 'logical'
```

3. Разрешим подключение пользователей к базе, в т.ч. для физической репликации

```
daa@daa-pgsql:~$ echo "host all all 0.0.0.0/0 md5" | sudo tee -a /etc/postgresql/15/main/pg_hba.conf
host all all 0.0.0.0/0 md5
daa@daa-pgsql:~$ echo "host replication all 0.0.0.0/0 md5" | sudo tee -a /etc/postgresql/15/main/pg_hba.conf
host replication all 0.0.0.0/0 md5
```

4. Перезапустим кластера 

```
sudo systemctl restart postgresql
```

5. Создадим пользователя 

```
daa@daa-pgsql:~$ sudo -u postgres psql -c "CREATE USER testrep SUPERUSER encrypted PASSWORD 'QWEasd123'"
CREATE ROLE
```

#### На 1 ВМ создаем таблицы test для записи, test2 для запросов на чтение.

```
daa@daa-pgsql:~$ sudo -u postgres psql -c "CREATE DATABASE repdb"
sudo -u postgres psql repdb -c "CREATE TABLE test1 (m text)"
sudo -u postgres psql repdb -c "CREATE TABLE test2 (m text)"
CREATE DATABASE
CREATE TABLE
CREATE TABLE
```

#### Создаем публикацию таблицы test и подписываемся на публикацию таблицы test2 с ВМ №2.

1. Создадим БД на ВМ 2

```
daa@daa-pgsql:~$ sudo -u postgres psql -c "CREATE DATABASE repdb"
CREATE DATABASE
```

2. Создадим таблицы на ВМ 2

```
daa@daa-pgsql:~$ sudo -u postgres psql repdb -c "CREATE TABLE test1 (m text)"
sudo -u postgres psql repdb -c "CREATE TABLE test2 (m text)"
CREATE TABLE
CREATE TABLE
```

3. Создадим публикацию на ВМ 1 для таблицы test

```
daa@daa-pgsql:~$ daa@daa-pgsql:~$ sudo -u postgres psql repdb -c "CREATE PUBLICATION test1_p FOR TABLE test1"
CREATE PUBLICATION
```

4. Подпишемся на публикацию таблицы test2 с ВМ 2

```
daa@daa-pgsql:~$ sudo -u postgres psql repdb -c "CREATE SUBSCRIPTION test1_sub CONNECTION 'host=192.168.1.106 port=5432 user=testrep password=QWEasd123 dbname=repdb' PUBLICATION test1_p WITH (copy_data = true)"
NOTICE:  created replication slot "test1_sub" on publisher
CREATE SUBSCRIPTION
```

#### Создаем публикацию таблицы test2 и подписываемся на публикацию таблицы test1 с ВМ №1.

1. Создадим публикацию таблицы test2 на ВМ 2

```
daa@daa-pgsql:~$ sudo -u postgres psql repdb -c "CREATE PUBLICATION test2_p FOR TABLE test2"
CREATE PUBLICATION
```

2. Подпишемся на публикацию таблицы test1 с ВМ 1

```
daa@daa-pgsql:~$ sudo -u postgres psql repdb -c "CREATE SUBSCRIPTION test2_sub CONNECTION 'host=192.168.1.107 port=5432 user=testrep password=QWEasd123 dbname=repdb' PUBLICATION test2_p WITH (copy_data = true)"
[sudo] password for daa:
NOTICE:  created replication slot "test2_sub" on publisher
CREATE SUBSCRIPTION
```

#### Проверим результат

1. Запишем данные в таблицу 1 ВМ 1

```
daa@daa-pgsql:~$ sudo -u postgres psql repdb -c "INSERT INTO test1 VALUES ('hello')"
INSERT 0 1
```

2. Запишем данные в таблицу 2 ВМ 2

```
daa@daa-pgsql:~$ sudo -u postgres psql repdb -c "INSERT INTO test2 VALUES ('world')"
INSERT 0 1
```

3. Проверим результат

ВМ 1

```
daa@daa-pgsql:~$ sudo -u postgres psql repdb -c "select * from test2"
   m
-------
 world
(1 row)
```

ВМ 2

```
daa@daa-pgsql:~$ sudo -u postgres psql repdb -c "select * from test1"
   m
-------
 hello
(1 row)
```

#### 3 ВМ использовать как реплику для чтения и бэкапов (подписаться на таблицы из ВМ №1 и №2 ).

1. Создадим БД на ВМ 3

```
daa@daa-pgsql:~$ sudo -u postgres psql -c "CREATE DATABASE repdb"
[sudo] password for daa:
CREATE DATABASE

```

2. Создадим таблицы на ВМ 3

```
daa@daa-pgsql:~$ sudo -u postgres psql repdb -c "CREATE TABLE test1 (m text)"
sudo -u postgres psql repdb -c "CREATE TABLE test2 (m text)"
CREATE TABLE
CREATE TABLE
```

3. Подпишемся на публикации для таблицы 1 в ВМ 1 и таблицы 2 в ВМ 2

```
daa@daa-pgsql:~$ sudo -u postgres psql repdb -c "CREATE SUBSCRIPTION test13_sub CONNECTION 'host=192.168.1.106 port=5432 user=testrep password=QWEasd123 dbname=repdb' PUBLICATION test1_p WITH (copy_data = true)"
sudo -u postgres psql repdb -c "CREATE SUBSCRIPTION test23_sub CONNECTION 'host=192.168.1.107 port=5432 user=testrep password=QWEasd123 dbname=repdb' PUBLICATION test2_p WITH (copy_data = true)"
NOTICE:  created replication slot "test13_sub" on publisher
CREATE SUBSCRIPTION
NOTICE:  created replication slot "test23_sub" on publisher
CREATE SUBSCRIPTION
```

Результат связей

```
daa@daa-pgsql:~$  sudo -u postgres psql repdb -c "\dRs"
could not change directory to "/home/daa": Permission denied
             List of subscriptions
    Name    |  Owner   | Enabled | Publication
------------+----------+---------+-------------
 test13_sub | postgres | t       | {test1_p}
 test23_sub | postgres | t       | {test2_p}
(2 rows)

daa@daa-pgsql:~$  sudo -u postgres psql repdb -c "select * from test2"
   m
-------
 world
(1 row)

daa@daa-pgsql:~$  sudo -u postgres psql repdb -c "select * from test1"
   m
-------
 hello
(1 row)
```

#### *Реализовать горячее реплицирование для высокой доступности на 4ВМ. Источником должна выступать ВМ №3. Написать с какими проблемами столкнулись.

Для этого используем pg_basebackup, т.к. утилита предназначена для создания базовых копий работающего кластера баз данных PostgreSQL. Процедура создания копии не влияет на работу других клиентов базы

1. Остановим кластер на ВМ 4

```
daa@daa-pgsql:~$ sudo systemctl stop postgresql
daa@daa-pgsql:~$ pg_lsclusters
Ver Cluster Port Status Owner    Data directory              Log file
15  main    5432 down   postgres /var/lib/postgresql/15/main /var/log/postgresql/postgresql-15-main.log
```

2. Удалим дефолтную БД

```
daa@daa-pgsql:~$ sudo -u postgres rm -rf /var/lib/postgresql/15/main/*
```

3. Создадим репликацию через pg_basepackup для ВМ 3

```
daa@daa-pgsql:~$ sudo -u postgres pg_basebackup --host=192.168.1.108 --port=5432 --username=testrep --pgdata=/var/lib/postgresql/15/main/ --progress --write-recovery-conf --create-slot --slot=replica1
Password:
30597/30597 kB (100%), 1/1 tablespace
```

4. Запустим кластер

```
daa@daa-pgsql:~$ sudo systemctl start postgresql
daa@daa-pgsql:~$ pg_lsclusters
Ver Cluster Port Status Owner    Data directory              Log file
15  main    5432 online postgres /var/lib/postgresql/15/main /var/log/postgresql/postgresql-15-main.log
```

5. Проверим результат

ВМ 1

```
daa@daa-pgsql:~$ sudo -u postgres psql repdb -c "INSERT INTO test1 VALUES ('hot?')"
[sudo] password for daa:
INSERT 0 1
```

ВМ 2

```
daa@daa-pgsql:~$ sudo -u postgres psql repdb -c "INSERT INTO test2 VALUES ('YES,very hot')"
[sudo] password for daa:
INSERT 0 1
```

ВМ 4

```
daa@daa-pgsql:~$ sudo -u postgres psql repdb -c "select * from test1"
could not change directory to "/home/daa": Permission denied
   m
-------
 hello
 hot?
(2 rows)

daa@daa-pgsql:~$ sudo -u postgres psql repdb -c "select * from test2"
could not change directory to "/home/daa": Permission denied
      m
--------------
 world
 YES,very hot
(2 rows)
```
