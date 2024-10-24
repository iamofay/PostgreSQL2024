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

#### Воспроизведем ситуацию при которой в журнале появится информация о блокировка

В сессии 1 запустим запрос на изменение данных в таблице

```
testlock=# begin;
BEGIN
testlock=*# update text_table SET text = 'QWE123asd' where id=1;
UPDATE 1
testlock=*# SELECT pg_backend_pid();
 pg_backend_pid
----------------
          22800
(1 строка)
```

В сессии 2 попробуем выполнить, например, автовакуум. Запрос завис в ожидании

```
testlock=# vacuum FULL text_table ;

```

В третьей сессии посмотрим журнал нашей БД postgresql-15-hw7.log, где уже увидим информацию о блокировке

```
daa@daa-VMware-Virtual-Platform:~$ tail -n 10 /var/log/postgresql/postgresql-15-hw7.log
2024-10-23 14:50:42.630 MSK [22870] postgres@testlock СООБЩЕНИЕ:  процесс 22870 продолжает ожидать в режиме AccessExclusiveLock блокировку "отношение 16390 базы данных 16388" в течение 204.056 мс
2024-10-23 14:50:42.630 MSK [22870] postgres@testlock ПОДРОБНОСТИ:  Process holding the lock: 22800. Wait queue: 22870.
2024-10-23 14:50:42.630 MSK [22870] postgres@testlock ОПЕРАТОР:  vacuum FULL text_table ;
```

#### Смоделируем ситуацию обновления одной и той же строки тремя командами UPDATE в разных сеансах.

Суссия 1

```
testlock=# begin;
BEGIN
testlock=*# update text_table SET text = 'QWE123asd1' where id=1;
UPDATE 1
```

Сессия 2

```
testlock=# begin;
BEGIN
testlock=*# update text_table SET text = 'QWE2' where id=1;

```

Сессия 3

```
testlock=# begin;
BEGIN
testlock=*# update text_table SET text = 'asd3' where id=1;

```

Сообщения в журнале

```
2024-10-23 15:07:13.328 MSK [22870] postgres@testlock СООБЩЕНИЕ:  процесс 22870 продолжает ожидать в режиме ShareLock блокировку "транзакция 747" в течение 214.302 мс
2024-10-23 15:07:13.328 MSK [22870] postgres@testlock ПОДРОБНОСТИ:  Process holding the lock: 22800. Wait queue: 22870.
2024-10-23 15:07:13.328 MSK [22870] postgres@testlock КОНТЕКСТ:  при изменении кортежа (6416,42) в отношении "text_table"
2024-10-23 15:07:13.328 MSK [22870] postgres@testlock ОПЕРАТОР:  update text_table SET text = 'QWE2' where id=1;
2024-10-23 15:07:43.352 MSK [23550] postgres@postgres ОШИБКА:  отношение "text_table" не существует (символ 8)
2024-10-23 15:07:43.352 MSK [23550] postgres@postgres ОПЕРАТОР:  update text_table SET text = 'asd3' where id=1;
2024-10-23 15:08:26.665 MSK [23557] postgres@testlock СООБЩЕНИЕ:  процесс 23557 продолжает ожидать в режиме ExclusiveLock блокировку "кортеж (6416,42) отношения 16390 базы данных 16388" в течение 212.105 мс
2024-10-23 15:08:26.665 MSK [23557] postgres@testlock ПОДРОБНОСТИ:  Process holding the lock: 22870. Wait queue: 23557.
2024-10-23 15:08:26.665 MSK [23557] postgres@testlock ОПЕРАТОР:  update text_table SET text = 'asd3' where id=1;
2024-10-23 15:26:53.330 MSK [22870] postgres@testlock ОПЕРАТОР:  update text_table SET text = 'QWE2' where id=1;
2024-10-23 15:26:53.330 MSK [23557] postgres@testlock СООБЩЕНИЕ:  процесс 23557 получил в режиме ExclusiveLock блокировку "кортеж (6416,42) отношения 16390 базы данных 16388" через 1106876.928 мс
2024-10-23 15:26:53.330 MSK [23557] postgres@testlock ОПЕРАТОР:  update text_table SET text = 'asd3' where id=1;
2024-10-23 15:26:53.540 MSK [23557] postgres@testlock СООБЩЕНИЕ:  процесс 23557 продолжает ожидать в режиме ShareLock блокировку "транзакция 747" в течение 210.085 мс
2024-10-23 15:26:53.540 MSK [23557] postgres@testlock ПОДРОБНОСТИ:  Process holding the lock: 22800. Wait queue: 23557.
2024-10-23 15:26:53.540 MSK [23557] postgres@testlock КОНТЕКСТ:  при изменении кортежа (6416,42) в отношении "text_table"
2024-10-23 15:26:53.540 MSK [23557] postgres@testlock ОПЕРАТОР:  update text_table SET text = 'asd3' where id=1;
2024-10-23 15:27:02.951 MSK [22870] postgres@testlock СООБЩЕНИЕ:  процесс 22870 продолжает ожидать в режиме ExclusiveLock блокировку "кортеж (6416,42) отношения 16390 базы данных 16388" в течение 212.854 мс
2024-10-23 15:27:02.951 MSK [22870] postgres@testlock ПОДРОБНОСТИ:  Process holding the lock: 23557. Wait queue: 22870.
2024-10-23 15:27:02.951 MSK [22870] postgres@testlock ОПЕРАТОР:  update text_table SET text = 'QWE2' where id=1;
```

Посмотрим блокировки в представлении pg_locks

```
testlock=# select locktype,database,relation,page,tuple,transactionid,virtualtransaction,pid,mode,granted,fastpath from pg_locks where pid in (22800,22870,23557)  order by pid ;
   locktype    | database | relation | page | tuple | transactionid | virtualtransaction |  pid  |       mode       | granted | fastpath
---------------+----------+----------+------+-------+---------------+--------------------+-------+------------------+---------+----------
 transactionid |          |          |      |       |           747 | 4/74               | 22800 | ExclusiveLock    | t       | f
 relation      |    16388 |    16396 |      |       |               | 4/74               | 22800 | RowExclusiveLock | t       | t
 relation      |    16388 |    16390 |      |       |               | 4/74               | 22800 | RowExclusiveLock | t       | t
 virtualxid    |          |          |      |       |               | 4/74               | 22800 | ExclusiveLock    | t       | t
 virtualxid    |          |          |      |       |               | 5/22               | 22870 | ExclusiveLock    | t       | t
 relation      |    16388 |    16390 |      |       |               | 5/22               | 22870 | RowExclusiveLock | t       | t
 transactionid |          |          |      |       |           750 | 5/22               | 22870 | ExclusiveLock    | t       | f
 tuple         |    16388 |    16390 | 6416 |    42 |               | 5/22               | 22870 | ExclusiveLock    | f       | f
 relation      |    16388 |    16396 |      |       |               | 5/22               | 22870 | RowExclusiveLock | t       | t
 tuple         |    16388 |    16390 | 6416 |    42 |               | 6/16               | 23557 | ExclusiveLock    | t       | f
 relation      |    16388 |    16390 |      |       |               | 6/16               | 23557 | RowExclusiveLock | t       | t
 virtualxid    |          |          |      |       |               | 6/16               | 23557 | ExclusiveLock    | t       | t
 transactionid |          |          |      |       |           747 | 6/16               | 23557 | ShareLock        | f       | f
 transactionid |          |          |      |       |           749 | 6/16               | 23557 | ExclusiveLock    | t       | f
 relation      |    16388 |    16396 |      |       |               | 6/16               | 23557 | RowExclusiveLock | t       | t
(15 строк)
```

Блокировки сесси 1

- 1 Блокировка транзакции в режиме ExclusiveLock (параллельно с этой транзакцией допускается только чтение)
- 2-3 Блокировка для изменения таблицы
- 4 блкировка витруального номера транзакции

```
 transactionid |          |          |      |       |           747 | 4/74               | 22800 | ExclusiveLock    | t       | f
 relation      |    16388 |    16396 |      |       |               | 4/74               | 22800 | RowExclusiveLock | t       | t
 relation      |    16388 |    16390 |      |       |               | 4/74               | 22800 | RowExclusiveLock | t       | t
 virtualxid    |          |          |      |       |               | 4/74               | 22800 | ExclusiveLock    | t       | t
```

Блокировки сесси 2

- 1 блокировка виртуального номера транзацкции
- 2, 5 блокировка для таблицы и ключа
- 3 блокировка транзакции
- 4 блокировка строки в режиме кортеж для обновления

```
 virtualxid    |          |          |      |       |               | 5/22               | 22870 | ExclusiveLock    | t       | t
 relation      |    16388 |    16390 |      |       |               | 5/22               | 22870 | RowExclusiveLock | t       | t
 transactionid |          |          |      |       |           750 | 5/22               | 22870 | ExclusiveLock    | t       | f
 tuple         |    16388 |    16390 | 6416 |    42 |               | 5/22               | 22870 | ExclusiveLock    | f       | f
 relation      |    16388 |    16396 |      |       |               | 5/22               | 22870 | RowExclusiveLock | t       | t
```

Блокировки сесси 3

- 1 блокировка строки в режиме кортеж для обновления
- 2, 6 блокировка для таблицы и ключа
- 3 блокировка виртуального номера транзацкции
- 4, 5 блокировка транзакции

```
tuple         |    16388 |    16390 | 6416 |    42 |               | 6/16               | 23557 | ExclusiveLock    | t       | f
 relation      |    16388 |    16390 |      |       |               | 6/16               | 23557 | RowExclusiveLock | t       | t
 virtualxid    |          |          |      |       |               | 6/16               | 23557 | ExclusiveLock    | t       | t
 transactionid |          |          |      |       |           747 | 6/16               | 23557 | ShareLock        | f       | f
 transactionid |          |          |      |       |           749 | 6/16               | 23557 | ExclusiveLock    | t       | f
 relation      |    16388 |    16396 |      |       |               | 6/16               | 23557 | RowExclusiveLock | t       | t
```

Или немного в другом виде

```
testlock=# SELECT locktype, mode, granted, pid, pg_blocking_pids(pid) AS wait_for FROM pg_locks WHERE relation = 'text_table'::regclass order by pid;
 locktype |       mode       | granted |  pid  | wait_for
----------+------------------+---------+-------+----------
 relation | RowExclusiveLock | t       | 22800 | {}
 relation | RowExclusiveLock | t       | 22870 | {23557}
 tuple    | ExclusiveLock    | f       | 22870 | {23557}
 relation | RowExclusiveLock | t       | 23557 | {22800}
 tuple    | ExclusiveLock    | t       | 23557 | {22800}
```

#### Воспроизведем взаимоблокировку трех транзакций.

Сессия 1

```
testlock=# begin;
BEGIN
testlock=*# update text_table SET text = 'QWE2' where id=1;
UPDATE 1
testlock=*# update text_table SET text = 'QWE2' where id=2;

```

Сессия 2

```
testlock=# begin;
BEGIN
testlock=*# update text_table SET text = 'QWE123asd1' where id=2;
UPDATE 1
testlock=*# update text_table SET text = 'QWE123asd1' where id=3;
UPDATE 1
```

Сессия 3

```
testlock=# begin;
BEGIN
testlock=*# update text_table SET text = 'asd3' where id=3;
UPDATE 1
testlock=*# update text_table SET text = 'asd3' where id=1;
ОШИБКА:  обнаружена взаимоблокировка
ПОДРОБНОСТИ:  Процесс 23557 ожидает в режиме ShareLock блокировку "транзакция 754"; заблокирован процессом 22870.
Процесс 22870 ожидает в режиме ShareLock блокировку "транзакция 755"; заблокирован процессом 22800.
Процесс 22800 ожидает в режиме ShareLock блокировку "транзакция 756"; заблокирован процессом 23557.
ПОДСКАЗКА:  Подробности запроса смотрите в протоколе сервера.
КОНТЕКСТ:  при изменении кортежа (6416,51) в отношении "text_table"
```

В журнале видно всю историю + получаем подробности в которых видна последовательность блокировки

```
2024-10-23 16:13:52.494 MSK [23557] postgres@testlock СООБЩЕНИЕ:  процесс 23557 получил в режиме ShareLock блокировку "транзакция 751" через 200184.404 мс
2024-10-23 16:13:52.494 MSK [23557] postgres@testlock КОНТЕКСТ:  при изменении кортежа (6416,46) в отношении "text_table"
2024-10-23 16:13:52.494 MSK [23557] postgres@testlock ОПЕРАТОР:  update text_table SET text = 'asd3' where id=1;
2024-10-23 16:14:13.141 MSK [13889] СООБЩЕНИЕ:  начата контрольная точка: time
2024-10-23 16:14:13.485 MSK [13889] СООБЩЕНИЕ:  контрольная точка завершена: записано буферов: 4 (0.0%); добавлено файлов WAL 0, удалено: 0, переработано: 0; запись=0.337 сек., синхр.=0.002 сек., всего=0.345 сек.; синхронизировано_файлов=3, самая_долгая_синхр.=0.001 сек., средняя=0.001 сек.; расстояние=19 kB, ожидалось=63289 kB
2024-10-23 16:14:37.684 MSK [22870] postgres@testlock СООБЩЕНИЕ:  процесс 22870 продолжает ожидать в режиме ShareLock блокировку "транзакция 755" в течение 202.535 мс
2024-10-23 16:14:37.684 MSK [22870] postgres@testlock ПОДРОБНОСТИ:  Process holding the lock: 22800. Wait queue: 22870.
2024-10-23 16:14:37.684 MSK [22870] postgres@testlock КОНТЕКСТ:  при изменении кортежа (6416,50) в отношении "text_table"
2024-10-23 16:14:37.684 MSK [22870] postgres@testlock ОПЕРАТОР:  update text_table SET text = 'QWE2' where id=2;
2024-10-23 16:14:44.813 MSK [22800] postgres@testlock СООБЩЕНИЕ:  процесс 22800 продолжает ожидать в режиме ShareLock блокировку "транзакция 756" в течение 214.091 мс
2024-10-23 16:14:44.813 MSK [22800] postgres@testlock ПОДРОБНОСТИ:  Process holding the lock: 23557. Wait queue: 22800.
2024-10-23 16:14:44.813 MSK [22800] postgres@testlock КОНТЕКСТ:  при изменении кортежа (6416,49) в отношении "text_table"
2024-10-23 16:14:44.813 MSK [22800] postgres@testlock ОПЕРАТОР:  update text_table SET text = 'QWE123asd1' where id=3;
2024-10-23 16:14:49.681 MSK [23557] postgres@testlock СООБЩЕНИЕ:  процесс 23557 обнаружил взаимоблокировку, ожидая в режиме ShareLock блокировку "транзакция 754" в течение 206.435 мс
2024-10-23 16:14:49.681 MSK [23557] postgres@testlock ПОДРОБНОСТИ:  Process holding the lock: 22870. Wait queue: .
2024-10-23 16:14:49.681 MSK [23557] postgres@testlock КОНТЕКСТ:  при изменении кортежа (6416,51) в отношении "text_table"
2024-10-23 16:14:49.681 MSK [23557] postgres@testlock ОПЕРАТОР:  update text_table SET text = 'asd3' where id=1;
2024-10-23 16:14:49.682 MSK [23557] postgres@testlock ОШИБКА:  обнаружена взаимоблокировка
2024-10-23 16:14:49.682 MSK [23557] postgres@testlock ПОДРОБНОСТИ:  Процесс 23557 ожидает в режиме ShareLock блокировку "транзакция 754"; заблокирован процессом 22870.
        Процесс 22870 ожидает в режиме ShareLock блокировку "транзакция 755"; заблокирован процессом 22800.
        Процесс 22800 ожидает в режиме ShareLock блокировку "транзакция 756"; заблокирован процессом 23557.
        Процесс 23557: update text_table SET text = 'asd3' where id=1;
        Процесс 22870: update text_table SET text = 'QWE2' where id=2;
        Процесс 22800: update text_table SET text = 'QWE123asd1' where id=3;
2024-10-23 16:14:49.682 MSK [23557] postgres@testlock ПОДСКАЗКА:  Подробности запроса смотрите в протоколе сервера.
2024-10-23 16:14:49.682 MSK [23557] postgres@testlock КОНТЕКСТ:  при изменении кортежа (6416,51) в отношении "text_table"
2024-10-23 16:14:49.682 MSK [23557] postgres@testlock ОПЕРАТОР:  update text_table SET text = 'asd3' where id=1;
2024-10-23 16:14:49.682 MSK [22800] postgres@testlock СООБЩЕНИЕ:  процесс 22800 получил в режиме ShareLock блокировку "транзакция 756" через 5082.980 мс
2024-10-23 16:14:49.682 MSK [22800] postgres@testlock КОНТЕКСТ:  при изменении кортежа (6416,49) в отношении "text_table"
2024-10-23 16:14:49.682 MSK [22800] postgres@testlock ОПЕРАТОР:  update text_table SET text = 'QWE123asd1' where id=3;
```

#### * Проверим, могут ли две транзакции, выполняющие единственную команду UPDATE одной и той же таблицы (без where), заблокировать друг друга?

Сессия 1

```
testlock=# BEGIN ISOLATION LEVEL REPEATABLE READ;
UPDATE text_table SET text= (select id from text_table order by id asc limit 1 for update);
COMMIT;
BEGIN
ОШИБКА:  обнаружена взаимоблокировка
ПОДРОБНОСТИ:  Процесс 25823 ожидает в режиме ShareLock блокировку "транзакция 762"; заблокирован процессом 25837.
Процесс 25837 ожидает в режиме ShareLock блокировку "транзакция 761"; заблокирован процессом 25823.
ПОДСКАЗКА:  Подробности запроса смотрите в протоколе сервера.
КОНТЕКСТ:  при изменении кортежа (10834,179) в отношении "text_table"
ROLLBACK
```

Сессия 2

```
testlock=# BEGIN ISOLATION LEVEL REPEATABLE READ;
UPDATE text_table SET text = (select id from text_table order by id desc limit 1 for update);
COMMIT;
BEGIN
UPDATE 1000000
COMMIT
```

Журнал блокировок

```
2024-10-24 09:10:48.530 MSK [25837] postgres@testlock СООБЩЕНИЕ:  процесс 25837 продолжает ожидать в режиме ShareLock блокировку "транзакция 761" в течение 200.782 мс
2024-10-24 09:10:48.530 MSK [25837] postgres@testlock ПОДРОБНОСТИ:  Process holding the lock: 25823. Wait queue: 25837.
2024-10-24 09:10:48.530 MSK [25837] postgres@testlock КОНТЕКСТ:  при изменении кортежа (6944,1) в отношении "text_table"
2024-10-24 09:10:48.530 MSK [25837] postgres@testlock ОПЕРАТОР:  UPDATE text_table SET text = (select id from text_table order by id desc limit 1 for update);
2024-10-24 09:10:53.228 MSK [25823] postgres@testlock СООБЩЕНИЕ:  процесс 25823 обнаружил взаимоблокировку, ожидая в режиме ShareLock блокировку "транзакция 762" в течение 200.531 мс
2024-10-24 09:10:53.228 MSK [25823] postgres@testlock ПОДРОБНОСТИ:  Process holding the lock: 25837. Wait queue: .
2024-10-24 09:10:53.228 MSK [25823] postgres@testlock КОНТЕКСТ:  при изменении кортежа (10834,179) в отношении "text_table"
2024-10-24 09:10:53.228 MSK [25823] postgres@testlock ОПЕРАТОР:  UPDATE text_table SET text= (select id from text_table order by id asc limit 1 for update);
2024-10-24 09:10:53.228 MSK [25823] postgres@testlock ОШИБКА:  обнаружена взаимоблокировка
2024-10-24 09:10:53.228 MSK [25823] postgres@testlock ПОДРОБНОСТИ:  Процесс 25823 ожидает в режиме ShareLock блокировку "транзакция 762"; заблокирован процессом 25837.
        Процесс 25837 ожидает в режиме ShareLock блокировку "транзакция 761"; заблокирован процессом 25823.
        Процесс 25823: UPDATE text_table SET text= (select id from text_table order by id asc limit 1 for update);
        Процесс 25837: UPDATE text_table SET text = (select id from text_table order by id desc limit 1 for update);
2024-10-24 09:10:53.228 MSK [25823] postgres@testlock ПОДСКАЗКА:  Подробности запроса смотрите в протоколе сервера.
2024-10-24 09:10:53.228 MSK [25823] postgres@testlock КОНТЕКСТ:  при изменении кортежа (10834,179) в отношении "text_table"
2024-10-24 09:10:53.228 MSK [25823] postgres@testlock ОПЕРАТОР:  UPDATE text_table SET text= (select id from text_table order by id asc limit 1 for update);
2024-10-24 09:10:53.228 MSK [25837] postgres@testlock СООБЩЕНИЕ:  процесс 25837 получил в режиме ShareLock блокировку "транзакция 761" через 4899.136 мс
2024-10-24 09:10:53.228 MSK [25837] postgres@testlock КОНТЕКСТ:  при изменении кортежа (6944,1) в отношении "text_table"
2024-10-24 09:10:53.228 MSK [25837] postgres@testlock ОПЕРАТОР:  UPDATE text_table SET text = (select id from text_table order by id desc limit 1 for update);
```

В данном случае мы пробуем забрать od для update в первом сеансе отсортированные по возрастанию, а во втором по убыванию. В тот момент когда запросы пересекаются возникает взаимная блокировка. В данном примере первым завершился запрос в перовм сеансе ROLLBACK, а дальше выполнился второй запрос.
