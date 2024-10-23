### Механизм блокировок

Цель:

- понимать как работает механизм блокировок объектов и строк.

#### Для выполнения работы будем использовать ВМ подготовленную для выполнения прошлых ДЗ

Для реализации домашнего задания использовалась VMWare Worstation, была поднята ВМ с ОС Ubuntu 20.04.1 desaktop

Настройте сервер так, чтобы в журнал сообщений сбрасывалась информация о блокировках, удерживаемых более 200 миллисекунд. Воспроизведите ситуацию, при которой в журнале появятся такие сообщения.

#### Развернем новый кластер и подключимся к нему. Создадим БД и таблицу с данными

```
daa@daa-VMware-Virtual-Platform:/usr/lib/postgresql/15/bin$ sudo pg_createcluster 15 hw7
Creating new PostgreSQL cluster 15/hw7 ...
/usr/lib/postgresql/15/bin/initdb -D /var/lib/postgresql/15/hw7 --auth-local peer --auth-host scram-sha-256 --no-instructions
Файлы, относящиеся к этой СУБД, будут принадлежать пользователю "postgres".
От его имени также будет запускаться процесс сервера.

Кластер баз данных будет инициализирован с локалью "ru_RU.UTF-8".
Кодировка БД по умолчанию, выбранная в соответствии с настройками: "UTF8".
Выбрана конфигурация текстового поиска по умолчанию "russian".

Контроль целостности страниц данных отключён.

исправление прав для существующего каталога /var/lib/postgresql/15/hw7... ок
создание подкаталогов... ок
выбирается реализация динамической разделяемой памяти... posix
выбирается значение max_connections по умолчанию... 100
выбирается значение shared_buffers по умолчанию... 128MB
выбирается часовой пояс по умолчанию... Europe/Moscow
создание конфигурационных файлов... ок
выполняется подготовительный скрипт... ок
выполняется заключительная инициализация... ок
сохранение данных на диске... ок
Ver Cluster Port Status Owner    Data directory             Log file
15  hw7     5432 down   postgres /var/lib/postgresql/15/hw7 /var/log/postgresql/postgresql-15-hw7.log
daa@daa-VMware-Virtual-Platform:/usr/lib/postgresql/15/bin$ sudo -u postgres pg_ctlcluster 15 hw7 start
Warning: the cluster will not be running as a systemd service. Consider using systemctl:
  sudo systemctl start postgresql@15-hw7
daa@daa-VMware-Virtual-Platform:/usr/lib/postgresql/15/bin$ sudo -u postgres psql
psql (17.0 (Ubuntu 17.0-1.pgdg24.04+1), сервер 15.8 (Ubuntu 15.8-1.pgdg24.04+1))
Введите "help", чтобы получить справку.

postgres=# CREATE DATABASE testlock;
CREATE DATABASE
postgres=# \c testlock;
psql (17.0 (Ubuntu 17.0-1.pgdg24.04+1), сервер 15.8 (Ubuntu 15.8-1.pgdg24.04+1))
Вы подключены к базе данных "testlock" как пользователь "postgres".
testlock=# SHOW lock_timeout;
 lock_timeout
--------------
 200ms
(1 строка)
testlock=# CREATE TABLE text_table (
    id SERIAL PRIMARY KEY,
    text TEXT
);
CREATE TABLE
testlock=# INSERT INTO text_table
SELECT id, random()::TEXT
FROM generate_series(1, 1000000) AS id;
INSERT 0 1000000
testlock=# select * from text_table limit 10;
 id |        text
----+---------------------
  1 | 0.4662269627169888
  2 | 0.3227542987939307
  3 | 0.8218730834434804
  4 | 0.9830776067709714
  5 | 0.44916795538743925
  6 | 0.8073347753165578
  7 | 0.8448333455807078
  8 | 0.3087115855309015
  9 | 0.3117268293531199
 10 | 0.7134107953357511
(10 строк)

```

#### Настроим сервер так, чтобы в журнал сообщений сбрасывалась информация о блокировках, удерживаемых более 200 миллисекунд. 

Для решения этой задачи можно исползовать параметр deadlock_timeout (integer) 

Время ожидания блокировки (в миллисекундах), по истечении которого будет выполняться проверка состояния взаимоблокировки.

```
testlock=#  alter system set deadlock_timeout to 200;
ALTER SYSTEM
testlock=# select pg_reload_conf();
 pg_reload_conf
----------------
 t
(1 строка)

testlock=# SHOW deadlock_timeout;
 deadlock_timeout
------------------
 200ms
(1 строка)
```

И не забудем включить журналирование log_lock_waits (boolean)

Определяет, нужно ли фиксировать в журнале события, когда сеанс ожидает получения блокировки дольше, чем указано в deadlock_timeout. Это позволяет выяснить, не связана ли низкая производительность с ожиданием блокировок. По умолчанию отключено. 

```
testlock=#  alter system set log_lock_waits to on;
ALTER SYSTEM
testlock=# select pg_reload_conf();
 pg_reload_conf
----------------
 t
(1 строка)

testlock=# select pg_reload_conf();
 pg_reload_conf
----------------
 t
(1 строка)

testlock=# SHOW log_lock_waits;
 log_lock_waits
----------------
 on
(1 строка)
```

#### Воспроизведем ситуацию при которой в журнале появится информация о блокировках


А так же необходимо включить журналирование 

Смоделируйте ситуацию обновления одной и той же строки тремя командами UPDATE в разных сеансах. Изучите возникшие блокировки в представлении pg_locks и убедитесь, что все они понятны. Пришлите список блокировок и объясните, что значит каждая.
Воспроизведите взаимоблокировку трех транзакций. Можно ли разобраться в ситуации постфактум, изучая журнал сообщений?
Могут ли две транзакции, выполняющие единственную команду UPDATE одной и той же таблицы (без where), заблокировать друг друга?
