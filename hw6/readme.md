### Работа с журналами

Цель:

- уметь работать с журналами и контрольными точками
- уметь настраивать параметры журналов

#### Для выполнения работы будем использовать ВМ подготовленную для выполнения прошлых ДЗ

Для реализации домашнего задания использовалась VMWare Worstation, была поднята ВМ с ОС Ubuntu 20.04.1 desaktop

Для тестирования будем использовать демо БД из документации postgresql

```
https://postgrespro.ru/docs/postgrespro/15/demodb-bookings-installation
```

```
daa@daa-VMware-Virtual-Platform:~$ psql -f demo_small_20241015.sql -U postgres
SET
SET
SET
...
ALTER TABLE
ALTER DATABASE
ALTER DATABASE
```

#### Настроить выполнение контрольной точки раз в 30 секунд.

Контрольные точки настраиваются в checkpoint_timeout (integer). По умолчанию оно равно пяти минутам (5min). Задать этот параметр можно только в postgresql.conf или в командной строке при запуске сервера. Зададим в конфиге

#### Изменим конфиг, перезапустим коластер 

```
daa@daa-VMware-Virtual-Platform:/etc/postgresql/15/main$ sudo nano postgresql.conf
```

![image](https://github.com/user-attachments/assets/1f286905-961b-4114-8a9e-097bfa24ef42)

```
daa@daa-VMware-Virtual-Platform:/etc/postgresql/15/main$ sudo -u postgres pg_ctlcluster 15 main stop
daa@daa-VMware-Virtual-Platform:/etc/postgresql/15/main$ sudo -u postgres pg_ctlcluster 15 main start
Warning: the cluster will not be running as a systemd service. Consider using systemctl:
  sudo systemctl start postgresql@15-main
daa@daa-VMware-Virtual-Platform:/etc/postgresql/15/main$ pg_conftool show all
checkpoint_timeout = 30s
```

#### Подадим нагрузку и замерим объем журналов за это время

Для начала посмотрим на крайний WAL файл

```
demo=# select pg_current_wal_lsn(), pg_current_wal_insert_lsn();
 pg_current_wal_lsn | pg_current_wal_insert_lsn
--------------------+---------------------------
 0/A31DCBF0         | 0/A31DCBF0
(1 строка)

demo=# SELECT * FROM pg_stat_bgwriter \gx
-[ RECORD 1 ]---------+------------------------------
checkpoints_timed     | 108
checkpoints_req       | 6
checkpoint_write_time | 1083498
checkpoint_sync_time  | 1161
buffers_checkpoint    | 65762
buffers_clean         | 1049
maxwritten_clean      | 10
buffers_backend       | 578657
buffers_backend_fsync | 0
buffers_alloc         | 107811
stats_reset           | 2024-10-15 14:27:41.678614+03
```

Теперь подадим нагрузку утилитой pgbench

```
daa@daa-VMware-Virtual-Platform:/etc/postgresql/15/main$ pgbench -c 8 -P 6 -T 600 -U postgres demo
pgbench (15.8 (Ubuntu 15.8-1.pgdg24.04+1))
starting vacuum...end.
progress: 6.0 s, 523.3 tps, lat 14.996 ms stddev 9.921, 0 failed
progress: 12.0 s, 524.3 tps, lat 15.223 ms stddev 22.894, 0 failed
...
progress: 600.0 s, 561.8 tps, lat 14.205 ms stddev 9.379, 0 failed
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 8
number of threads: 1
maximum number of tries: 1
duration: 600 s
number of transactions actually processed: 340081
number of failed transactions: 0 (0.000%)
latency average = 14.082 ms
latency stddev = 9.812 ms
initial connection time = 90.559 ms
tps = 566.863519 (without initial connection time)
```

За период нагрузки был пройден 21 чекпойнт, все чейкпойнты выполнились в установленных рамках в соответствии с расписанием.

```
demo=# SELECT * FROM pg_stat_bgwriter \gx
-[ RECORD 1 ]---------+------------------------------
checkpoints_timed     | 129
checkpoints_req       | 6
checkpoint_write_time | 1595413
checkpoint_sync_time  | 1346
buffers_checkpoint    | 104097
buffers_clean         | 1049
maxwritten_clean      | 10
buffers_backend       | 580873
buffers_backend_fsync | 0
buffers_alloc         | 110019
stats_reset           | 2024-10-15 14:27:41.678614+03

demo=# select pg_current_wal_lsn(), pg_current_wal_insert_lsn();
 pg_current_wal_lsn | pg_current_wal_insert_lsn
--------------------+---------------------------
 0/BE427F20         | 0/BE427F20
(1 строка)
```

Теперь посмотрим объем файлов

```
demo=# select '0/A31DCBF0'::pg_lsn - '0/BE427F20'::pg_lsn as bytes;
   bytes
------------
 -455390000
(1 строка)
```

Сделаем более информативно

```
demo=# select pg_size_pretty('0/A31DCBF0'::pg_lsn - '0/BE427F20'::pg_lsn);
 pg_size_pretty
----------------
 -434 MB
(1 строка)
```

Соответсвенно объем одного файла примерно 21 Mb 

```
demo=# select pg_size_pretty(('0/A31DCBF0'::pg_lsn - '0/BE427F20'::pg_lsn) / 21);
 pg_size_pretty
----------------
 -21 MB
(1 строка)
```

#### Проверим данные статистики: все ли контрольные точки выполнялись точно по расписанию. Почему так произошло?

Все чейкпойнты выполнились в установленных рамках в соответствии с расписанием.

```
demo=# SELECT * FROM pg_stat_bgwriter \gx
-[ RECORD 1 ]---------+------------------------------
checkpoints_timed     | 129
checkpoints_req       | 6
checkpoint_write_time | 1595413
checkpoint_sync_time  | 1346
buffers_checkpoint    | 104097
buffers_clean         | 1049
maxwritten_clean      | 10
buffers_backend       | 580873
buffers_backend_fsync | 0
buffers_alloc         | 110019
stats_reset           | 2024-10-15 14:27:41.678614+03
```

В настройках max_wal_size установлен в 1GB, соответственно мы не смогли достичь предела, поэтому все отработало корректно

```
daa@daa-VMware-Virtual-Platform:/etc/postgresql/15/main$ pg_conftool show all
checkpoint_timeout = 30s
cluster_name = '15/main'
data_directory = '/var/lib/postgresql/15/main'
datestyle = 'iso, dmy'
default_text_search_config = 'pg_catalog.russian'
dynamic_shared_memory_type = posix
external_pid_file = '/var/run/postgresql/15-main.pid'
hba_file = '/etc/postgresql/15/main/pg_hba.conf'
ident_file = '/etc/postgresql/15/main/pg_ident.conf'
lc_messages = 'ru_RU.UTF-8'
lc_monetary = 'ru_RU.UTF-8'
lc_numeric = 'ru_RU.UTF-8'
lc_time = 'ru_RU.UTF-8'
log_line_prefix = '%m [%p] %q%u@%d '
log_timezone = 'Europe/Moscow'
max_connections = 100
max_wal_size = 1GB
min_wal_size = 80MB
port = 5432
shared_buffers = 128MB
ssl = on
ssl_cert_file = '/etc/ssl/certs/ssl-cert-snakeoil.pem'
ssl_key_file = '/etc/ssl/private/ssl-cert-snakeoil.key'
timezone = 'Europe/Moscow'
unix_socket_directories = '/var/run/postgresql'
```

#### Сравним tps в синхронном/асинхронном режиме утилитой pgbench. Объясним полученный результат.

Включим асинхронный режим

```
demo=# ALTER SYSTEM SET synchronous_commit TO OFF;
ALTER SYSTEM
demo=# show synchronous_commit;
 synchronous_commit
--------------------
 off
(1 строка)
```

Исходя из документации:

Асинхронная фиксация — это возможность завершать транзакции быстрее, ценой того, что в случае краха СУБД последние транзакции могут быть потеряны.
Подтверждение транзакции обычно синхронное: сервер ждёт сохранения записей WAL транзакции в постоянном хранилище, прежде чем сообщить клиенту об успешном завершении. Таким образом, клиенту гарантируется, что транзакция, которую подтвердил сервер, будет защищена, даже если сразу после этого произойдёт крах сервера. Однако для коротких транзакций данная задержка будет основной составляющей общего времени транзакции. В режиме асинхронного подтверждения сервер сообщает об успешном завершении сразу, как только транзакция будет завершена логически, прежде чем сгенерированные записи WAL фактически будут записаны на диск. Это может значительно увеличить производительность при выполнении небольших транзакций.

Получим ожидаемый результат, tps в асинхронном режиме гораздо выше

```
daa@daa-VMware-Virtual-Platform:/etc/postgresql/15/main$ pgbench -c 8 -P 6 -T 600 -U postgres demo
pgbench (15.8 (Ubuntu 15.8-1.pgdg24.04+1))
starting vacuum...end.
progress: 6.0 s, 2153.0 tps, lat 3.697 ms stddev 2.615, 0 failed
progress: 12.0 s, 2352.8 tps, lat 3.394 ms stddev 2.418, 0 failed
...
progress: 594.0 s, 1126.2 tps, lat 7.086 ms stddev 5.282, 0 failed
progress: 600.0 s, 995.3 tps, lat 8.025 ms stddev 6.387, 0 failed
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 8
number of threads: 1
maximum number of tries: 1
duration: 600 s
number of transactions actually processed: 966875
number of failed transactions: 0 (0.000%)
latency average = 4.956 ms
latency stddev = 4.291 ms
initial connection time = 19.213 ms
tps = 1611.477166 (without initial connection time)
```

#### Создадим новый кластер с включенной контрольной суммой страниц.

```
daa@daa-VMware-Virtual-Platform:/usr/lib/postgresql/15/bin$ sudo pg_createcluste                                                r 15 testdb -- --data-checksums
Creating new PostgreSQL cluster 15/testdb ...
/usr/lib/postgresql/15/bin/initdb -D /var/lib/postgresql/15/testdb --auth-local                                                 peer --auth-host scram-sha-256 --no-instructions --data-checksums
Файлы, относящиеся к этой СУБД, будут принадлежать пользователю "postgres".
От его имени также будет запускаться процесс сервера.

Кластер баз данных будет инициализирован с локалью "ru_RU.UTF-8".
Кодировка БД по умолчанию, выбранная в соответствии с настройками: "UTF8".
Выбрана конфигурация текстового поиска по умолчанию "russian".

Контроль целостности страниц данных включён.

исправление прав для существующего каталога /var/lib/postgresql/15/testdb... ок
создание подкаталогов... ок
выбирается реализация динамической разделяемой памяти... posix
выбирается значение max_connections по умолчанию... 100
выбирается значение shared_buffers по умолчанию... 128MB
выбирается часовой пояс по умолчанию... Europe/Moscow
создание конфигурационных файлов... ок
выполняется подготовительный скрипт... ок
выполняется заключительная инициализация... ок
сохранение данных на диске... ок
Ver Cluster Port Status Owner    Data directory                Log file
15  testdb  5432 down   postgres /var/lib/postgresql/15/testdb /var/log/postgres                                                ql/postgresql-15-testdb.log
daa@daa-VMware-Virtual-Platform:/usr/lib/postgresql/15/bin$ sudo -u postgres pg_ctlcluster 15 testdb start
Warning: the cluster will not be running as a systemd service. Consider using systemctl:
  sudo systemctl start postgresql@15-testdb
daa@daa-VMware-Virtual-Platform:/usr/lib/postgresql/15/bin$ sudo -u postgres pg_lsclusters
Ver Cluster Port Status Owner    Data directory                Log file
15  testdb  5432 online postgres /var/lib/postgresql/15/testdb /var/log/postgresql/postgresql-15-testdb.log
```

Проверим, что контрольные суммы включены

```
daa@daa-VMware-Virtual-Platform:/usr/lib/postgresql/15/bin$ sudo -u postgres psql
psql (17.0 (Ubuntu 17.0-1.pgdg24.04+1), сервер 15.8 (Ubuntu 15.8-1.pgdg24.04+1))
Введите "help", чтобы получить справку.

postgres=# SHOW data_checksums;
 data_checksums
----------------
 on
(1 строка)
```

#### Создадим таблицу. Вставим в нее несклько значений.

```
postgres=# CREATE TABLE text_table (
    id SERIAL PRIMARY KEY,
    text TEXT
);
CREATE TABLE
postgres=# INSERT INTO text_table
SELECT id, random()::TEXT
FROM generate_series(1, 10) AS id;
INSERT 0 10
```

#### Выключим кластер. 

```
postgres=# \q
daa@daa-VMware-Virtual-Platform:/usr/lib/postgresql/15/bin$ sudo -u postgres pg_ctlcluster 15 testdb stop
daa@daa-VMware-Virtual-Platform:/usr/lib/postgresql/15/bin$ sudo -u postgres pg_lsclusters
Ver Cluster Port Status Owner    Data directory                Log file
15  testdb  5432 down   postgres /var/lib/postgresql/15/testdb /var/log/postgresql/postgresql-15-testdb.log
```


Измените пару байт в таблице. Включите кластер и сделайте выборку из таблицы. Что и почему произошло? как проигнорировать ошибку и продолжить работу?
