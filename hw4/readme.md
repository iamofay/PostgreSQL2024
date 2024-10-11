### Работа с базами данных, пользователями и правами

Цель:

- создание новой базы данных, схемы и таблицы
- создание роли для чтения данных из созданной схемы созданной базы данных
- создание роли для чтения и записи из созданной схемы созданной базы данных

#### Cоздать новый проект в Яндекс облако или на любых ВМ, докере.

Для реализации домашнего задания использовалась VMWare Worstation, была поднята ВМ с ОС Ubuntu 20.04.1 desaktop

#### Установка OpenSSH

```
sudo apt install openssh-server
```

![image](https://github.com/user-attachments/assets/7620c019-f33a-4e16-9775-e477b5bfd14d)

#### Включим SSH

```
sudo systemctl enable --now ssh
```
![image](https://github.com/user-attachments/assets/0d7ad1d9-e9aa-49cf-b921-94e7d1bc11c4)

#### Проверим статус

```
sudo systemctl status ssh
```
![image](https://github.com/user-attachments/assets/43adbe0b-e8db-49bf-8d5f-ff71b677dae6)

#### Сгенирируем ключ SSH в PuttyGen

![image](https://github.com/user-attachments/assets/afa38444-4ffb-4a91-8f2d-748630c4f31d)

#### Добавим публичный ключ на нашем сервере

```
echo ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABAQCcBdtfxlpr+fKkhoTxxIAYJoXV5vLlrkztykJn2gFkmDDt6Gon0EUsSY0lk10C/SzwoVmfQR3GYEtpGbeGex8b0SaSxf4K/K6GUJgsBprbasabwYpA37P2PTE9Y7fIxrLsl4PjhWbDnalysl/Qef/LTUgiH6mR1oUxYSa+HP3D/i1L0O5XNI6jw88G6a+eKZlloVbcCxkSsC1+1Ay7NjAirZUR6BeQpUEnzM/KCm3nlx3z/adYQSZlb6i36ZibK4w5N3+o2wU0GKRyCrlWKnMo+fLtWQbZyp+O2DoN9+fYRiQ4ggfI1bgDXLT6UPQdGG3QxuB5Sqs4RFvI4NdmCgTH rsa-key-20241007 >> ~/.ssh/authorized_keys
```

#### Свяжем пользователя и ключ

```
chown -R daa:daa ~/.ssh
```

#### Добавим закрыйтый ключ в оснастке putty клиентского устройства

![image](https://github.com/user-attachments/assets/d812510c-24ad-4416-a21a-3f8cff3e066e)

#### Пробуем подключиться и успешно подключаемся без пароля

![image](https://github.com/user-attachments/assets/53d020ab-e080-4c15-807e-d6abb86d74b6)

#### Установим PostgreSQL 14

```
daa@daa-VMware-Virtual-Platform:~$ sudo apt install postgresql-client-14 postgresql-14
daa@daa-VMware-Virtual-Platform:~$ sudo -u postgres pg_lsclusters
Ver Cluster Port Status Owner    Data directory              Log file
14  main    5432 online postgres /var/lib/postgresql/14/main /var/log/postgresql/postgresql-14-main.log
```

#### Подключимся к PostgreSQL

```
daa@daa-VMware-Virtual-Platform:~$ sudo -u postgres psql
psql (17.0 (Ubuntu 17.0-1.pgdg24.04+1), сервер 14.13 (Ubuntu 14.13-1.pgdg24.04+1))
Введите "help", чтобы получить справку.

postgres=#
```

#### Создадим и подключимся к новой БД testnm

```
postgres=# CREATE DATABASE testdb;
CREATE DATABASE
postgres=# \c testdb
psql (17.0 (Ubuntu 17.0-1.pgdg24.04+1), сервер 14.13 (Ubuntu 14.13-1.pgdg24.04+1))
Вы подключены к базе данных "testdb" как пользователь "postgres".
```

#### Создадим схему testnm

```
testdb=# CREATE SCHEMA testnm;
CREATE SCHEMA
```

#### Создадим новую таблицу t1 с одной колонкой c1 типа integer

```
testdb=# CREATE TABLE t1(c1 integer);
CREATE TABLE
```

#### Создадим новую таблицу t1 с одной колонкой c1 типа integer и вставим строку со значением c1=1

```
testdb=# INSERT INTO t1(c1) VALUES(1);
INSERT 0 1
testdb=# select * from t1
;
 c1
----
  1
(1 строка)
```

#### Создадим новую роль readonly

```
testdb=# CREATE role readonly;
CREATE ROLE
```

#### Дадим новой роли право на подключение к базе данных testdb

```
testdb=# grant connect on DATABASE testdb TO readonly;
GRANT
```

#### Дадим новой роли право на select для всех таблиц схемы testnm

```
testdb=# grant select on all tables in schema testnm to readonly;
GRANT
```

#### Создадим пользователя testread с паролем test123 и ролью readonly

```
testdb=# create user testread with password 'test123' in role readonly;
CREATE ROLE
```

#### При попытке подключиться новым пользователем к базе получим ошибку

```
postgres=# \c testdb testread
подключиться к серверу через сокет "/var/run/postgresql/.s.PGSQL.5432" не удалось: ВАЖНО:  пользователь "testread" не прошёл проверку подлинности (Peer)
Сохранено предыдущее подключение
```

#### Исправим проблему в pg_hba.conf, после внесения изменений перезапустим кластер, чтобы изменения применились

```
daa@daa-VMware-Virtual-Platform:~$ cd /etc/postgresql/14/main/
daa@daa-VMware-Virtual-Platform:/etc/postgresql/14/main$ sudo nano pg_hba.conf  
```

![image](https://github.com/user-attachments/assets/44c5a2f1-dafe-42e4-b132-e93153566f0a)

#### Успешно подключимся пользователем 

```
daa@daa-VMware-Virtual-Platform:/etc/postgresql/14/main$ sudo -u postgres psql
psql (17.0 (Ubuntu 17.0-1.pgdg24.04+1), сервер 14.13 (Ubuntu 14.13-1.pgdg24.04+1))
Введите "help", чтобы получить справку.

postgres=# \c testdb testread
Пароль пользователя testread:
psql (17.0 (Ubuntu 17.0-1.pgdg24.04+1), сервер 14.13 (Ubuntu 14.13-1.pgdg24.04+1))
Вы подключены к базе данных "testdb" как пользователь "testread".
testdb=>
```

#### Попробуем обратиться к таблице select * from t1;

```
testdb=> select * from t1;
ОШИБКА:  нет доступа к таблице t1
```

Закономерно получили ошибку доступа, т.к. таблица t1 находится в схеме public, на которую прав мы не давали. Но мы это исправим, дадим прав для схемы.

```
testdb=> \c testdb postgres
psql (17.0 (Ubuntu 17.0-1.pgdg24.04+1), сервер 14.13 (Ubuntu 14.13-1.pgdg24.04+1))
Вы подключены к базе данных "testdb" как пользователь "postgres".
testdb=# grant select on all tables in schema public to readonly;
GRANT
testdb=# \c testdb testread
Пароль пользователя testread:
psql (17.0 (Ubuntu 17.0-1.pgdg24.04+1), сервер 14.13 (Ubuntu 14.13-1.pgdg24.04+1))
Вы подключены к базе данных "testdb" как пользователь "testread".
testdb=> select * from t1;
 c1
----
  1
(1 строка)
```

Теперь все получилось.

#### Вернемся в пользователя postgres

```
testdb=> \c testdb postgres
psql (17.0 (Ubuntu 17.0-1.pgdg24.04+1), сервер 14.13 (Ubuntu 14.13-1.pgdg24.04+1))
Вы подключены к базе данных "testdb" как пользователь "postgres".
testdb=#
```

#### Удалим таблицу t1

```
testdb=# drop table t1;
DROP TABLE
```



#### Создадим новую таблицу t1 с явным указанием схемы testnm

```
testdb=# CREATE TABLE testnm.t1(c1 integer);
CREATE TABLE
testdb=# INSERT INTO testnm.t1(c1) VALUES(1);
INSERT 0 1
testdb=# select * from testnm.t1
;
 c1
----
  1
(1 строка)
```

#### Создадим новую таблицу t1 с одной колонкой c1 типа integer и вставим строку со значением c1=1

```
testdb=# INSERT INTO testnm.t1(c1) VALUES(1);
INSERT 0 1
testdb=# select * from testnm.t1
;
 c1
----
  1
(1 строка)
```

Связано это с тем, что права были выданы пользователю для тех таблиц, которые уже были созданы на момент определения прав.
Для того, чтобы это исправить необходимо прибегнуть к ALTER DEFAULT PRIVILEGES, назначению прав по умолчанию

#### Дадим прав по дефолту для роли readonly

```
testdb=> \c testdb postgres
psql (17.0 (Ubuntu 17.0-1.pgdg24.04+1), сервер 14.13 (Ubuntu 14.13-1.pgdg24.04+1))
Вы подключены к базе данных "testdb" как пользователь "postgres".
testdb=# alter default privileges in schema testnm grant select on tables to readonly;
ALTER DEFAULT PRIVILEGES
```

#### Попробуем создать новую таблицу t3 в схеме testnm

```
testdb=# CREATE TABLE testnm.t3(c1 integer);
CREATE TABLE
testdb=# INSERT INTO testnm.t3(c1) VALUES(1);
INSERT 0 1
```

#### Проверим результат под пользователем testread

```
testdb=# \c testdb testread
Пароль пользователя testread:
psql (17.0 (Ubuntu 17.0-1.pgdg24.04+1), сервер 14.13 (Ubuntu 14.13-1.pgdg24.04+1))
Вы подключены к базе данных "testdb" как пользователь "testread".
testdb=> select * from testnm.t3;
 c1
----
  1
(1 строка)
```

Теперь для всех создаваемых таблиц члены роли readonly будут иметь парава на select, в том числе пользователь testread

#### Попробуем выполнить команду create table t2(c1 integer); insert into t2 values (2); под пользователем testread

```
testdb=> create table t2(c1 integer);
CREATE TABLE
testdb=> insert into t2 values (2);
INSERT 0 1
```

Получили закономерный успех, т.к. в соответствии с документацией https://postgrespro.ru/docs/postgresql/14/ddl-schemas все по умолчаню имеют доступ CREATE и USAGE в public, но при условии, что у них есть права на подключение.

![image](https://github.com/user-attachments/assets/b3a8317c-b90a-4c2b-bc1c-8b16c52b504e)

#### Такую дыру лучше прикрыть

```
testdb=> \c testdb postgres
psql (17.0 (Ubuntu 17.0-1.pgdg24.04+1), сервер 14.13 (Ubuntu 14.13-1.pgdg24.04+1))
Вы подключены к базе данных "testdb" как пользователь "postgres".
testdb=# REVOKE CREATE ON SCHEMA public FROM PUBLIC;
REVOKE
testdb=# \c testdb testread
Пароль пользователя testread:
psql (17.0 (Ubuntu 17.0-1.pgdg24.04+1), сервер 14.13 (Ubuntu 14.13-1.pgdg24.04+1))
Вы подключены к базе данных "testdb" как пользователь "testread".
testdb=> create table t3(c1 integer);
ОШИБКА:  нет доступа к схеме public
СТРОКА 1: create table t3(c1 integer);
```

Теперь прав на создание таблиц в схеме public нет

#### Такую дыру лучше прикрыть

```
testdb=> \c testdb postgres
psql (17.0 (Ubuntu 17.0-1.pgdg24.04+1), сервер 14.13 (Ubuntu 14.13-1.pgdg24.04+1))
Вы подключены к базе данных "testdb" как пользователь "postgres".
testdb=# REVOKE CREATE ON SCHEMA public FROM PUBLIC;
REVOKE
testdb=# \c testdb testread
Пароль пользователя testread:
psql (17.0 (Ubuntu 17.0-1.pgdg24.04+1), сервер 14.13 (Ubuntu 14.13-1.pgdg24.04+1))
Вы подключены к базе данных "testdb" как пользователь "testread".
testdb=> create table t3(c1 integer);
ОШИБКА:  нет доступа к схеме public
СТРОКА 1: create table t3(c1 integer);
                       ^
```

#### Однако INSERT в уже созданную таблицу работает, это можно решить удалением таблица или забрав права на USAGE

```
testdb=# REVOKE USAGE ON SCHEMA public FROM PUBLIC;
REVOKE
testdb=# \c testdb testread
Пароль пользователя testread:
psql (17.0 (Ubuntu 17.0-1.pgdg24.04+1), сервер 14.13 (Ubuntu 14.13-1.pgdg24.04+1))
Вы подключены к базе данных "testdb" как пользователь "testread".
testdb=> insert into t2 values (2);
ОШИБКА:  отношение "t2" не существует
СТРОКА 1: insert into t2 values (2);
                      ^
testdb=> insert into public.t2 values (2);
ОШИБКА:  нет доступа к схеме public
СТРОКА 1: insert into public.t2 values (2);
                      ^
```

Но чтобы такого не было, лучше заранее подготовить роли в БД и отключить INSTERT в Public для всех пользователей.
