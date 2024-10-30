### Нагрузочное тестирование и тюнинг PostgreSQL

Цель:

- сделать нагрузочное тестирование PostgreSQL
- настроить параметры PostgreSQL для достижения максимальной производительности

#### Для выполнения работы будем использовать ВМ подготовленную для выполнения прошлых ДЗ

Для реализации домашнего задания использовалась VMWare Worstation, была поднята ВМ с ОС Ubuntu 20.04.1 desaktop

![image](https://github.com/user-attachments/assets/6b5c173c-6317-4da9-aae5-174931cff7ed)

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

#### Посмотрим на настройки по умолчанию

```
daa@daa-VMware-Virtual-Platform:~$ sudo -u postgres psql
[sudo] пароль для daa:
Попробуйте ещё раз.
[sudo] пароль для daa:
psql (17.0 (Ubuntu 17.0-1.pgdg24.04+1), сервер 15.8 (Ubuntu 15.8-1.pgdg24.04+1))
Введите "help", чтобы получить справку.

postgres=# \c demo
psql (17.0 (Ubuntu 17.0-1.pgdg24.04+1), сервер 15.8 (Ubuntu 15.8-1.pgdg24.04+1))
Вы подключены к базе данных "demo" как пользователь "postgres".
demo=# select sourcefile,name,setting,applied from pg_file_settings;
               sourcefile                |            name            |                setting                 | applied
-----------------------------------------+----------------------------+----------------------------------------+---------
 /etc/postgresql/15/main/postgresql.conf | data_directory             | /var/lib/postgresql/15/main            | t
 /etc/postgresql/15/main/postgresql.conf | hba_file                   | /etc/postgresql/15/main/pg_hba.conf    | t
 /etc/postgresql/15/main/postgresql.conf | ident_file                 | /etc/postgresql/15/main/pg_ident.conf  | t
 /etc/postgresql/15/main/postgresql.conf | external_pid_file          | /var/run/postgresql/15-main.pid        | t
 /etc/postgresql/15/main/postgresql.conf | port                       | 5432                                   | t
 /etc/postgresql/15/main/postgresql.conf | max_connections            | 100                                    | t
 /etc/postgresql/15/main/postgresql.conf | unix_socket_directories    | /var/run/postgresql                    | t
 /etc/postgresql/15/main/postgresql.conf | ssl                        | on                                     | t
 /etc/postgresql/15/main/postgresql.conf | ssl_cert_file              | /etc/ssl/certs/ssl-cert-snakeoil.pem   | t
 /etc/postgresql/15/main/postgresql.conf | ssl_key_file               | /etc/ssl/private/ssl-cert-snakeoil.key | t
 /etc/postgresql/15/main/postgresql.conf | shared_buffers             | 128MB                                  | t
 /etc/postgresql/15/main/postgresql.conf | dynamic_shared_memory_type | posix                                  | t
 /etc/postgresql/15/main/postgresql.conf | max_wal_size               | 1GB                                    | t
 /etc/postgresql/15/main/postgresql.conf | min_wal_size               | 80MB                                   | t
 /etc/postgresql/15/main/postgresql.conf | log_line_prefix            | %m [%p] %q%u@%d                        | t
 /etc/postgresql/15/main/postgresql.conf | log_timezone               | Europe/Moscow                          | t
 /etc/postgresql/15/main/postgresql.conf | cluster_name               | 15/main                                | t
 /etc/postgresql/15/main/postgresql.conf | datestyle                  | iso, dmy                               | t
 /etc/postgresql/15/main/postgresql.conf | timezone                   | Europe/Moscow                          | t
 /etc/postgresql/15/main/postgresql.conf | lc_messages                | ru_RU.UTF-8                            | t
 /etc/postgresql/15/main/postgresql.conf | lc_monetary                | ru_RU.UTF-8                            | t
 /etc/postgresql/15/main/postgresql.conf | lc_numeric                 | ru_RU.UTF-8                            | t
 /etc/postgresql/15/main/postgresql.conf | lc_time                    | ru_RU.UTF-8                            | t
 /etc/postgresql/15/main/postgresql.conf | default_text_search_config | pg_catalog.russian                     | t
(24 строки)
```

#### Нагрузим БД в текущей конфигурации с помошью pgbench

```
daa@daa-VMware-Virtual-Platform:~$ pgbench -c 99 -P 6 -T 60 -U postgres demo
pgbench (15.8 (Ubuntu 15.8-1.pgdg24.04+1))
starting vacuum...end.
progress: 6.0 s, 1075.7 tps, lat 84.703 ms stddev 107.463, 0 failed
progress: 12.0 s, 1122.3 tps, lat 87.593 ms stddev 126.281, 0 failed
progress: 18.0 s, 1107.8 tps, lat 90.152 ms stddev 117.145, 0 failed
progress: 24.0 s, 1344.2 tps, lat 73.285 ms stddev 95.274, 0 failed
progress: 30.0 s, 1135.3 tps, lat 87.142 ms stddev 118.158, 0 failed
progress: 36.0 s, 1088.2 tps, lat 90.712 ms stddev 114.475, 0 failed
progress: 42.0 s, 1211.4 tps, lat 81.723 ms stddev 118.048, 0 failed
progress: 48.0 s, 1185.1 tps, lat 83.913 ms stddev 115.802, 0 failed
progress: 54.0 s, 1218.9 tps, lat 81.027 ms stddev 107.126, 0 failed
progress: 60.0 s, 1072.3 tps, lat 92.316 ms stddev 134.786, 0 failed
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 99
number of threads: 1
maximum number of tries: 1
duration: 60 s
number of transactions actually processed: 69466
number of failed transactions: 0 (0.000%)
latency average = 85.081 ms
latency stddev = 115.676 ms
initial connection time = 383.375 ms
tps = 1161.581572 (without initial connection time)
```

#### Воспользуемся ресурсом https://www.pgconfig.org/ для оределения оптимальной конфигурации

Предлагается следующий конфиг

```
# Memory Configuration
shared_buffers = 1GB # Увеличить буфер, т.к. память быстрее диска
effective_cache_size = 3GB # влияет на оценку стоимости использования индекса; чем выше это значение, тем больше вероятность, что будет применяться сканирование по индексу. По умолчанию 4GB
work_mem = 10MB # Задаёт объём памяти, который будет использоваться для внутренних операций сортировки и хеш-таблиц, прежде чем будут задействованы временные файлы на диске. Значение по умолчанию — четыре мегабайта (4MB)
maintenance_work_mem = 205MB # Задаёт максимальный объём памяти для операций обслуживания БД, в частности VACUUM, CREATE INDEX и ALTER TABLE ADD FOREIGN KEY. По умолчанию его значение — 64 мегабайта (64MB).

# Checkpoint Related Configuration
min_wal_size = 2GB # Минимальный рамзер WAL. Значение по умолчанию — 1 ГБ. 
max_wal_size = 3GB # Максимальный размер WAL. Значение по умолчанию — 80 МБ
checkpoint_completion_target = 0.9 # Доля общего времени между контрольными точками. Значение по умолчанию — 0.9
wal_buffers = -1 # Объём разделяемой памяти, который будет использоваться для буферизации данных WAL, ещё не записанных на диск. Значение по умолчанию, равное -1, задаёт размер, равный 1/32 (около 3%) от shared_buffers, но не меньше чем 64 КБ и не больше чем размер одного сегмента WAL (обычно 16 МБ)

# Network Related Configuration
listen_addresses = '*' # Задаёт адреса TCP/IP, по которым сервер будет принимать подключения клиентских приложений. Особый элемент, *, обозначает все имеющиеся IP-интерфейсы
max_connections = 100 # Определяет максимальное число одновременных подключений к серверу БД. По умолчанию 100

# Storage Configuration
random_page_cost = 1.1 # Задаёт приблизительную стоимость чтения одной произвольной страницы с диска. Значение по умолчанию равно 4.0.
effective_io_concurrency = 200 # Задаёт допустимое число параллельных операций ввода/вывода, которое говорит PostgreSQL о том, сколько операций ввода/вывода могут быть выполнены одновременно.

# Worker Processes Configuration
max_worker_processes = 8 # Задаёт максимальное число фоновых процессов, которое можно запустить в текущей системе. Значение по умолчанию — 8.
max_parallel_workers_per_gather = 2 # Задаёт максимальное число рабочих процессов, которые могут запускаться одним узлом Gather или Gather Merge. Значение по умолчанию — 2.
max_parallel_workers = 2 # Задаёт максимальное число рабочих процессов, которое система сможет поддерживать для параллельных операций. Значение по умолчанию — 8

# Logging configuration for pgbadger
logging_collector = on # Это фоновый процесс, который собирает отправленные в stderr сообщения и перенаправляет их в журнальные файлы.
log_checkpoints = on # Включает протоколирование выполнения контрольных точек и точек перезапуска сервера.
log_connections = on # Включает протоколирование всех попыток подключения к серверу, в том числе успешного завершения как аутентификации (если она требуется), так и авторизации клиентов. Значение по умолчанию — off.
log_disconnections = on # Включает протоколирование завершения сеанса. Значение по умолчанию — off.
log_lock_waits = on # Определяет, нужно ли фиксировать в журнале события, когда сеанс ожидает получения блокировки дольше, чем указано в deadlock_timeout. По умолчанию отключён. 
log_temp_files = 0 # Управляет регистрацией в журнале имён и размеров временных файлов. При нулевом значении записывается информация обо всех файлах, а при положительном — о файлах, размер которых не меньше заданной величины. Значение по умолчанию равно -1, то есть запись такой информации отключена.
lc_messages = 'C' # Устанавливает язык выводимых сообщений.

# Adjust the minimum time to collect the data
log_min_duration_statement = '10s' # Записывает в журнал продолжительность выполнения всех команд, время работы которых не меньше указанного. Со значением -1 (по умолчанию) запись полностью отключается.
log_autovacuum_min_duration = 0 # Задаёт время выполнения действия автоочистки, при превышении которого информация об этом действии записывается в журнал. При нулевом значении в журнале фиксируются все действия автоочистки.

# CSV Configuration
log_destination = 'csvlog' # Добавление csvlog в log_destination делает удобным загрузку журнальных файлов в таблицу базы данных.
```

#### Применим конфигурацию

```
demo=# select sourcefile,name,setting,applied from pg_file_settings;
                    sourcefile                    |              name               |                setting                 | applied
--------------------------------------------------+---------------------------------+----------------------------------------+---------
 /etc/postgresql/15/main/postgresql.conf          | data_directory                  | /var/lib/postgresql/15/main            | t
 /etc/postgresql/15/main/postgresql.conf          | hba_file                        | /etc/postgresql/15/main/pg_hba.conf    | t
 /etc/postgresql/15/main/postgresql.conf          | ident_file                      | /etc/postgresql/15/main/pg_ident.conf  | t
 /etc/postgresql/15/main/postgresql.conf          | external_pid_file               | /var/run/postgresql/15-main.pid        | t
 /etc/postgresql/15/main/postgresql.conf          | port                            | 5432                                   | t
 /etc/postgresql/15/main/postgresql.conf          | max_connections                 | 100                                    | f
 /etc/postgresql/15/main/postgresql.conf          | unix_socket_directories         | /var/run/postgresql                    | t
 /etc/postgresql/15/main/postgresql.conf          | ssl                             | on                                     | t
 /etc/postgresql/15/main/postgresql.conf          | ssl_cert_file                   | /etc/ssl/certs/ssl-cert-snakeoil.pem   | t
 /etc/postgresql/15/main/postgresql.conf          | ssl_key_file                    | /etc/ssl/private/ssl-cert-snakeoil.key | t
 /etc/postgresql/15/main/postgresql.conf          | shared_buffers                  | 128MB                                  | f
 /etc/postgresql/15/main/postgresql.conf          | dynamic_shared_memory_type      | posix                                  | t
 /etc/postgresql/15/main/postgresql.conf          | max_wal_size                    | 1GB                                    | f
 /etc/postgresql/15/main/postgresql.conf          | min_wal_size                    | 80MB                                   | f
 /etc/postgresql/15/main/postgresql.conf          | log_line_prefix                 | %m [%p] %q%u@%d                        | t
 /etc/postgresql/15/main/postgresql.conf          | log_timezone                    | Europe/Moscow                          | t
 /etc/postgresql/15/main/postgresql.conf          | cluster_name                    | 15/main                                | t
 /etc/postgresql/15/main/postgresql.conf          | datestyle                       | iso, dmy                               | t
 /etc/postgresql/15/main/postgresql.conf          | timezone                        | Europe/Moscow                          | t
 /etc/postgresql/15/main/postgresql.conf          | lc_messages                     | ru_RU.UTF-8                            | f
 /etc/postgresql/15/main/postgresql.conf          | lc_monetary                     | ru_RU.UTF-8                            | t
 /etc/postgresql/15/main/postgresql.conf          | lc_numeric                      | ru_RU.UTF-8                            | t
 /etc/postgresql/15/main/postgresql.conf          | lc_time                         | ru_RU.UTF-8                            | t
 /etc/postgresql/15/main/postgresql.conf          | default_text_search_config      | pg_catalog.russian                     | t
 /var/lib/postgresql/15/main/postgresql.auto.conf | shared_buffers                  | 1GB                                    | f
 /var/lib/postgresql/15/main/postgresql.auto.conf | effective_cache_size            | 3GB                                    | t
 /var/lib/postgresql/15/main/postgresql.auto.conf | work_mem                        | 10MB                                   | t
 /var/lib/postgresql/15/main/postgresql.auto.conf | maintenance_work_mem            | 205MB                                  | t
 /var/lib/postgresql/15/main/postgresql.auto.conf | min_wal_size                    | 2GB                                    | t
 /var/lib/postgresql/15/main/postgresql.auto.conf | max_wal_size                    | 3GB                                    | t
 /var/lib/postgresql/15/main/postgresql.auto.conf | checkpoint_completion_target    | 0.9                                    | t
 /var/lib/postgresql/15/main/postgresql.auto.conf | wal_buffers                     | -1                                     | t
 /var/lib/postgresql/15/main/postgresql.auto.conf | listen_addresses                | *                                      | f
 /var/lib/postgresql/15/main/postgresql.auto.conf | max_connections                 | 100                                    | t
 /var/lib/postgresql/15/main/postgresql.auto.conf | random_page_cost                | 1.1                                    | t
 /var/lib/postgresql/15/main/postgresql.auto.conf | effective_io_concurrency        | 200                                    | t
 /var/lib/postgresql/15/main/postgresql.auto.conf | max_worker_processes            | 8                                      | t
 /var/lib/postgresql/15/main/postgresql.auto.conf | max_parallel_workers_per_gather | 2                                      | t
 /var/lib/postgresql/15/main/postgresql.auto.conf | max_parallel_workers            | 2                                      | t
 /var/lib/postgresql/15/main/postgresql.auto.conf | logging_collector               | on                                     | f
 /var/lib/postgresql/15/main/postgresql.auto.conf | log_checkpoints                 | on                                     | t
 /var/lib/postgresql/15/main/postgresql.auto.conf | log_connections                 | on                                     | t
 /var/lib/postgresql/15/main/postgresql.auto.conf | log_disconnections              | on                                     | t
 /var/lib/postgresql/15/main/postgresql.auto.conf | log_lock_waits                  | on                                     | t
 /var/lib/postgresql/15/main/postgresql.auto.conf | log_temp_files                  | 0                                      | t
 /var/lib/postgresql/15/main/postgresql.auto.conf | lc_messages                     | C                                      | t
 /var/lib/postgresql/15/main/postgresql.auto.conf | log_min_duration_statement      | 10s                                    | t
 /var/lib/postgresql/15/main/postgresql.auto.conf | log_autovacuum_min_duration     | 0                                      | t
 /var/lib/postgresql/15/main/postgresql.auto.conf | log_destination                 | csvlog                                 | t
(49 строк)
```

#### Проверим результат

```
daa@daa-VMware-Virtual-Platform:~$ pgbench -c 99 -P 6 -T 60 -U postgres demo
pgbench (15.8 (Ubuntu 15.8-1.pgdg24.04+1))
starting vacuum...end.
progress: 6.0 s, 1011.0 tps, lat 87.918 ms stddev 104.259, 0 failed
progress: 12.0 s, 1179.9 tps, lat 84.317 ms stddev 114.149, 0 failed
progress: 18.0 s, 1063.5 tps, lat 93.684 ms stddev 125.544, 0 failed
progress: 24.0 s, 1202.5 tps, lat 82.022 ms stddev 106.790, 0 failed
progress: 30.0 s, 1193.3 tps, lat 83.274 ms stddev 104.148, 0 failed
progress: 36.0 s, 1050.7 tps, lat 92.437 ms stddev 120.765, 0 failed
progress: 42.0 s, 1043.1 tps, lat 96.510 ms stddev 130.469, 0 failed
progress: 48.0 s, 1106.5 tps, lat 88.491 ms stddev 112.329, 0 failed
progress: 54.0 s, 1075.0 tps, lat 92.620 ms stddev 124.261, 0 failed
progress: 60.0 s, 988.9 tps, lat 100.632 ms stddev 132.782, 0 failed
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 99
number of threads: 1
maximum number of tries: 1
duration: 60 s
number of transactions actually processed: 65585
number of failed transactions: 0 (0.000%)
latency average = 90.003 ms
latency stddev = 117.772 ms
initial connection time = 448.786 ms
tps = 1098.226916 (without initial connection time)
```

Предыдущий результат, видим что стало даже хуже на 60 tps

```
tps = 1161.581572 (without initial connection time)
```

#### Теперь попробуем изменить конфигурацию не обращая внимания на проблемы с надежностью


synchronous_commit = off # Отключим ожидание для подтверждения успешности транзакции
fsync = off # Отключим синхронизацию записи изменений на диск
wal_level = minimal # Снизим количество информации записываемое в WAL
checkpoint_timeout = 30min # Увеличим период чекпойнтов, чтобы они не мешали в ходе теста
full_page_writes = off # Отключим записи страниц при изменениях после контрольной точки
max_wal_senders = 0 # Отключим процессы передачи WAL
data_checksums # Отключим контроль целостности данных

```
demo=# select sourcefile,name,setting,applied from pg_file_settings;
                    sourcefile                    |              name               |                setting                 | applied
--------------------------------------------------+---------------------------------+----------------------------------------+---------
 /etc/postgresql/15/main/postgresql.conf          | data_directory                  | /var/lib/postgresql/15/main            | t
 /etc/postgresql/15/main/postgresql.conf          | hba_file                        | /etc/postgresql/15/main/pg_hba.conf    | t
 /etc/postgresql/15/main/postgresql.conf          | ident_file                      | /etc/postgresql/15/main/pg_ident.conf  | t
 /etc/postgresql/15/main/postgresql.conf          | external_pid_file               | /var/run/postgresql/15-main.pid        | t
 /etc/postgresql/15/main/postgresql.conf          | port                            | 5432                                   | t
 /etc/postgresql/15/main/postgresql.conf          | max_connections                 | 100                                    | f
 /etc/postgresql/15/main/postgresql.conf          | unix_socket_directories         | /var/run/postgresql                    | t
 /etc/postgresql/15/main/postgresql.conf          | ssl                             | on                                     | t
 /etc/postgresql/15/main/postgresql.conf          | ssl_cert_file                   | /etc/ssl/certs/ssl-cert-snakeoil.pem   | t
 /etc/postgresql/15/main/postgresql.conf          | ssl_key_file                    | /etc/ssl/private/ssl-cert-snakeoil.key | t
 /etc/postgresql/15/main/postgresql.conf          | shared_buffers                  | 128MB                                  | f
 /etc/postgresql/15/main/postgresql.conf          | dynamic_shared_memory_type      | posix                                  | t
 /etc/postgresql/15/main/postgresql.conf          | max_wal_size                    | 1GB                                    | f
 /etc/postgresql/15/main/postgresql.conf          | min_wal_size                    | 80MB                                   | f
 /etc/postgresql/15/main/postgresql.conf          | log_line_prefix                 | %m [%p] %q%u@%d                        | t
 /etc/postgresql/15/main/postgresql.conf          | log_timezone                    | Europe/Moscow                          | t
 /etc/postgresql/15/main/postgresql.conf          | cluster_name                    | 15/main                                | t
 /etc/postgresql/15/main/postgresql.conf          | datestyle                       | iso, dmy                               | t
 /etc/postgresql/15/main/postgresql.conf          | timezone                        | Europe/Moscow                          | t
 /etc/postgresql/15/main/postgresql.conf          | lc_messages                     | ru_RU.UTF-8                            | f
 /etc/postgresql/15/main/postgresql.conf          | lc_monetary                     | ru_RU.UTF-8                            | t
 /etc/postgresql/15/main/postgresql.conf          | lc_numeric                      | ru_RU.UTF-8                            | t
 /etc/postgresql/15/main/postgresql.conf          | lc_time                         | ru_RU.UTF-8                            | t
 /etc/postgresql/15/main/postgresql.conf          | default_text_search_config      | pg_catalog.russian                     | t
 /var/lib/postgresql/15/main/postgresql.auto.conf | shared_buffers                  | 1GB                                    | t
 /var/lib/postgresql/15/main/postgresql.auto.conf | effective_cache_size            | 3GB                                    | t
 /var/lib/postgresql/15/main/postgresql.auto.conf | work_mem                        | 10MB                                   | t
 /var/lib/postgresql/15/main/postgresql.auto.conf | maintenance_work_mem            | 205MB                                  | t
 /var/lib/postgresql/15/main/postgresql.auto.conf | min_wal_size                    | 2GB                                    | t
 /var/lib/postgresql/15/main/postgresql.auto.conf | max_wal_size                    | 3GB                                    | t
 /var/lib/postgresql/15/main/postgresql.auto.conf | checkpoint_completion_target    | 0.9                                    | t
 /var/lib/postgresql/15/main/postgresql.auto.conf | wal_buffers                     | -1                                     | t
 /var/lib/postgresql/15/main/postgresql.auto.conf | listen_addresses                | *                                      | t
 /var/lib/postgresql/15/main/postgresql.auto.conf | max_connections                 | 100                                    | t
 /var/lib/postgresql/15/main/postgresql.auto.conf | random_page_cost                | 1.1                                    | t
 /var/lib/postgresql/15/main/postgresql.auto.conf | effective_io_concurrency        | 200                                    | t
 /var/lib/postgresql/15/main/postgresql.auto.conf | max_worker_processes            | 8                                      | t
 /var/lib/postgresql/15/main/postgresql.auto.conf | max_parallel_workers_per_gather | 2                                      | t
 /var/lib/postgresql/15/main/postgresql.auto.conf | max_parallel_workers            | 2                                      | t
 /var/lib/postgresql/15/main/postgresql.auto.conf | log_temp_files                  | 0                                      | t
 /var/lib/postgresql/15/main/postgresql.auto.conf | lc_messages                     | C                                      | t
 /var/lib/postgresql/15/main/postgresql.auto.conf | log_min_duration_statement      | 10s                                    | t
 /var/lib/postgresql/15/main/postgresql.auto.conf | log_autovacuum_min_duration     | 0                                      | t
 /var/lib/postgresql/15/main/postgresql.auto.conf | log_destination                 | csvlog                                 | t
 /var/lib/postgresql/15/main/postgresql.auto.conf | synchronous_commit              | off                                    | t
 /var/lib/postgresql/15/main/postgresql.auto.conf | fsync                           | off                                    | t
 /var/lib/postgresql/15/main/postgresql.auto.conf | wal_level                       | minimal                                | f
 /var/lib/postgresql/15/main/postgresql.auto.conf | checkpoint_timeout              | 30min                                  | t
 /var/lib/postgresql/15/main/postgresql.auto.conf | full_page_writes                | off                                    | t
 /var/lib/postgresql/15/main/postgresql.auto.conf | max_wal_senders                 | 0                                      | f
 /var/lib/postgresql/15/main/postgresql.auto.conf | logging_collector               | off                                    | f
 /var/lib/postgresql/15/main/postgresql.auto.conf | log_checkpoints                 | off                                    | t
 /var/lib/postgresql/15/main/postgresql.auto.conf | log_connections                 | off                                    | t
 /var/lib/postgresql/15/main/postgresql.auto.conf | log_disconnections              | off                                    | t
 /var/lib/postgresql/15/main/postgresql.auto.conf | log_lock_waits                  | off                                    | t
(55 строк)
```

data_checksums # Проверим, что отключен

```
demo=# SHOW data_checksums;
 data_checksums
----------------
 off
(1 строка)
```

#### Повторим тесты

```
daa@daa-VMware-Virtual-Platform:~$ pgbench -c 99 -P 6 -T 60 -U postgres demo
pgbench (15.8 (Ubuntu 15.8-1.pgdg24.04+1))
starting vacuum...end.
progress: 6.0 s, 1165.7 tps, lat 77.972 ms stddev 98.846, 0 failed
progress: 12.0 s, 1352.5 tps, lat 72.733 ms stddev 97.937, 0 failed
progress: 18.0 s, 1262.0 tps, lat 78.804 ms stddev 104.369, 0 failed
progress: 24.0 s, 1211.1 tps, lat 80.727 ms stddev 108.609, 0 failed
progress: 30.0 s, 997.3 tps, lat 100.418 ms stddev 126.794, 0 failed
progress: 36.0 s, 1161.7 tps, lat 85.185 ms stddev 112.215, 0 failed
progress: 42.0 s, 1291.0 tps, lat 76.847 ms stddev 104.522, 0 failed
progress: 48.0 s, 1260.5 tps, lat 78.283 ms stddev 104.268, 0 failed
progress: 54.0 s, 1327.4 tps, lat 74.520 ms stddev 94.890, 0 failed
progress: 60.0 s, 1264.4 tps, lat 78.445 ms stddev 99.417, 0 failed
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 99
number of threads: 1
maximum number of tries: 1
duration: 60 s
number of transactions actually processed: 73860
number of failed transactions: 0 (0.000%)
latency average = 79.983 ms
latency stddev = 105.251 ms
initial connection time = 390.989 ms
tps = 1235.524863 (without initial connection time)
```

Предыдущий результат, видим что стало получше на 70 tps

```
tps = 1161.581572 (without initial connection time)
```


#### * Протестируем через утилиту https://github.com/Percona-Lab/sysbench-tpcc

- Установим sysnench https://github.com/akopytov/sysbench

- Добавим пользователя, создадим БД и наделим пользователя правами

postgres=# CREATE USER sbtest WITH PASSWORD 'password';
CREATE ROLE
postgres=# CREATE DATABASE sbtest;
CREATE DATABASE
postgres=# GRANT ALL PRIVILEGES ON DATABASE sbtest TO sbtest;
GRANT
postgres=# GRANT ALL PRIVILEGES ON SCHEMA public TO sbtest;
GRANT

- Изменим hba конфиг

![image](https://github.com/user-attachments/assets/950a836e-93b3-4838-be31-da9b3d2d0afe)

- Подготовим БД

```
daa@daa-VMware-Virtual-Platform:~$ sysbench --db-driver=pgsql --report-interval=5 --threads=40 --time=60 --pgsql-host=localhost --pgsql-port=5432 --pgsql-user=sbtest --pgsql-password=password --pgsql-db=sbtest /usr/share/sysbench/oltp_read_write.lua cleanup
sysbench 1.0.20 (using system LuaJIT 2.1.0-beta3)

Dropping table 'sbtest1'...
daa@daa-VMware-Virtual-Platform:~$ sysbench --db-driver=pgsql --report-interval=5 --threads=40 --time=60 --pgsql-host=localhost --pgsql-port=5432 --pgsql-user=sbtest --pgsql-password=password --pgsql-db=sbtest /usr/share/sysbench/oltp_read_write.lua prepare
sysbench 1.0.20 (using system LuaJIT 2.1.0-beta3)

Initializing worker threads...

Creating table 'sbtest1'...
Inserting 10000 records into 'sbtest1'
Creating a secondary index on 'sbtest1'...
```

- Запустим тест

До изменеиня конфигурации

```
daa@daa-VMware-Virtual-Platform:~$ sysbench --db-driver=pgsql --report-interval=5 --threads=40 --time=60 --pgsql-host=localhost --pgsql-port=5432 --pgsql-user=sbtest --pgsql-password=password --pgsql-db=sbtest /usr/share/sysbench/oltp_read_write.lua run
sysbench 1.0.20 (using system LuaJIT 2.1.0-beta3)

Running the test with following options:
Number of threads: 40
Report intermediate results every 5 second(s)
Initializing random number generator from current time


Initializing worker threads...

Threads started!

[ 5s ] thds: 40 tps: 163.81 qps: 3563.88 (r/w/o: 2533.87/660.84/369.17) lat (ms,95%): 1089.30 err/s: 9.19 reconn/s: 0.00
[ 10s ] thds: 40 tps: 391.10 qps: 8114.14 (r/w/o: 5702.30/1573.81/838.02) lat (ms,95%): 92.42 err/s: 16.20 reconn/s: 0.00
[ 15s ] thds: 40 tps: 358.56 qps: 7396.68 (r/w/o: 5196.29/1429.26/771.12) lat (ms,95%): 943.16 err/s: 12.60 reconn/s: 0.00
[ 20s ] thds: 40 tps: 199.80 qps: 4161.50 (r/w/o: 2928.87/799.42/433.21) lat (ms,95%): 995.51 err/s: 9.40 reconn/s: 0.00
[ 25s ] thds: 40 tps: 334.32 qps: 6904.54 (r/w/o: 4854.03/1333.48/717.03) lat (ms,95%): 1013.60 err/s: 12.40 reconn/s: 0.00
[ 30s ] thds: 40 tps: 345.10 qps: 7141.53 (r/w/o: 5015.70/1376.21/749.62) lat (ms,95%): 960.30 err/s: 14.80 reconn/s: 0.00
[ 35s ] thds: 40 tps: 701.53 qps: 14386.87 (r/w/o: 10099.28/2807.70/1479.89) lat (ms,95%): 92.42 err/s: 18.21 reconn/s: 0.00
[ 40s ] thds: 40 tps: 117.61 qps: 2490.32 (r/w/o: 1755.68/476.02/258.61) lat (ms,95%): 1050.76 err/s: 7.80 reconn/s: 0.00
[ 45s ] thds: 40 tps: 172.87 qps: 3582.13 (r/w/o: 2520.52/694.08/367.53) lat (ms,95%): 1050.76 err/s: 7.19 reconn/s: 0.00
[ 50s ] thds: 40 tps: 134.41 qps: 2808.98 (r/w/o: 1977.33/537.23/294.42) lat (ms,95%): 2045.74 err/s: 6.80 reconn/s: 0.00
[ 55s ] thds: 40 tps: 199.02 qps: 4151.86 (r/w/o: 2920.72/796.29/434.85) lat (ms,95%): 1013.60 err/s: 9.60 reconn/s: 0.00
[ 60s ] thds: 40 tps: 111.20 qps: 2337.59 (r/w/o: 1649.19/445.20/243.20) lat (ms,95%): 1032.01 err/s: 6.60 reconn/s: 0.00
SQL statistics:
    queries performed:
        read:                            235844
        write:                           64772
        other:                           34840
        total:                           335456
    transactions:                        16186  (263.05 per sec.)
    queries:                             335456 (5451.70 per sec.)
    ignored errors:                      660    (10.73 per sec.)
    reconnects:                          0      (0.00 per sec.)

General statistics:
    total time:                          61.5307s
    total number of events:              16186

Latency (ms):
         min:                                    1.17
         avg:                                  150.94
         max:                                 6076.64
         95th percentile:                      995.51
         sum:                              2443076.80

Threads fairness:
    events (avg/stddev):           404.6500/16.53
    execution time (avg/stddev):   61.0769/0.50
```

После изменения конфигурации

```
daa@daa-VMware-Virtual-Platform:~$ sysbench --db-driver=pgsql --report-interval=5 --threads=40 --time=60 --pgsql-host=localhost --pgsql-port=5432 --pgsql-user=sbtest --pgsql-password=password --pgsql-db=sbtest /usr/share/sysbench/oltp_read_write.lua run
sysbench 1.0.20 (using system LuaJIT 2.1.0-beta3)

Running the test with following options:
Number of threads: 40
Report intermediate results every 5 second(s)
Initializing random number generator from current time


Initializing worker threads...

Threads started!

[ 5s ] thds: 40 tps: 390.78 qps: 8218.64 (r/w/o: 5798.22/1573.32/847.10) lat (ms,95%): 153.02 err/s: 15.38 reconn/s: 0.00
[ 10s ] thds: 40 tps: 80.00 qps: 1696.64 (r/w/o: 1198.43/321.61/176.60) lat (ms,95%): 2120.76 err/s: 5.60 reconn/s: 0.00
[ 15s ] thds: 40 tps: 419.58 qps: 8654.03 (r/w/o: 6081.34/1681.13/891.56) lat (ms,95%): 82.96 err/s: 14.80 reconn/s: 0.00
[ 20s ] thds: 40 tps: 215.80 qps: 4490.36 (r/w/o: 3158.37/868.99/463.00) lat (ms,95%): 1032.01 err/s: 9.80 reconn/s: 0.00
[ 25s ] thds: 40 tps: 447.47 qps: 9223.44 (r/w/o: 6477.26/1786.26/959.91) lat (ms,95%): 995.51 err/s: 16.99 reconn/s: 0.00
[ 30s ] thds: 40 tps: 434.21 qps: 8959.70 (r/w/o: 6298.18/1738.25/923.28) lat (ms,95%): 89.16 err/s: 14.21 reconn/s: 0.00
[ 35s ] thds: 40 tps: 610.59 qps: 12454.36 (r/w/o: 8735.43/2426.35/1292.58) lat (ms,95%): 90.78 err/s: 15.20 reconn/s: 0.00
[ 40s ] thds: 40 tps: 284.45 qps: 5932.97 (r/w/o: 4177.93/1140.18/614.87) lat (ms,95%): 960.30 err/s: 11.79 reconn/s: 0.00
[ 45s ] thds: 40 tps: 308.71 qps: 6339.81 (r/w/o: 4448.95/1230.25/660.61) lat (ms,95%): 995.51 err/s: 11.40 reconn/s: 0.00
[ 50s ] thds: 40 tps: 176.86 qps: 3696.75 (r/w/o: 2606.61/711.22/378.92) lat (ms,95%): 1973.38 err/s: 7.00 reconn/s: 0.00
[ 55s ] thds: 40 tps: 171.01 qps: 3548.51 (r/w/o: 2494.88/685.02/368.61) lat (ms,95%): 1069.86 err/s: 7.20 reconn/s: 0.00
[ 60s ] thds: 40 tps: 271.97 qps: 5655.68 (r/w/o: 3978.43/1090.10/587.15) lat (ms,95%): 995.51 err/s: 12.20 reconn/s: 0.00
SQL statistics:
    queries performed:
        read:                            277368
        write:                           76396
        other:                           40876
        total:                           394640
    transactions:                        19099  (305.39 per sec.)
    queries:                             394640 (6310.15 per sec.)
    ignored errors:                      713    (11.40 per sec.)
    reconnects:                          0      (0.00 per sec.)

General statistics:
    total time:                          62.5365s
    total number of events:              19099

Latency (ms):
         min:                                    1.09
         avg:                                  129.66
         max:                                 5063.71
         95th percentile:                      995.51
         sum:                              2476355.08

Threads fairness:
    events (avg/stddev):           477.4750/14.62
    execution time (avg/stddev):   61.9089/0.81



```

#### Выводы по sysbench

Однозначно наблюдается прирост производительности по всем параметрам, причем прирост более заметный, чем в случае с утилитой pgbench.
