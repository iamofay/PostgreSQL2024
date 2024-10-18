### Настройка autovacuum с учетом особеностей производительности

Цель:

- запустить нагрузочный тест pgbench
- настроить параметры autovacuum
- проверить работу autovacuum

#### Создать инстанс ВМ с 2 ядрами и 4 Гб ОЗУ и SSD 10GB

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

#### Дополнительно попробуем отдельно протестировать добавление nvme диска для ВМ, т.к. на используемом ноутбуке и так стоит SSD. Для тестирования будем использовать демо БД из документации postgresql
#### https://postgrespro.ru/docs/postgrespro/15/demodb-bookings-installation
#### Для начала попробуем без добавления nvme

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

#### Выполним pgbench -i postgres

```
daa@daa-VMware-Virtual-Platform:~$ pgbench -i -U postgres postgres
dropping old tables...
ЗАМЕЧАНИЕ:  таблица "pgbench_accounts" не существует, пропускается
ЗАМЕЧАНИЕ:  таблица "pgbench_branches" не существует, пропускается
ЗАМЕЧАНИЕ:  таблица "pgbench_history" не существует, пропускается
ЗАМЕЧАНИЕ:  таблица "pgbench_tellers" не существует, пропускается
creating tables...
generating data (client-side)...
100000 of 100000 tuples (100%) done (elapsed 0.12 s, remaining 0.00 s)
vacuuming...
creating primary keys...
done in 0.29 s (drop tables 0.00 s, create tables 0.03 s, client-side generate 0  
```

#### Теперь pgbench -c8 -P 6 -T 60 -U postgres postgres. Но выполним операцию 2 раза

1 раз
```
daa@daa-VMware-Virtual-Platform:~$ pgbench -c 8 -P 6 -T 60 -U postgres demo
pgbench (15.8 (Ubuntu 15.8-1.pgdg24.04+1))
starting vacuum...end.
progress: 6.0 s, 453.6 tps, lat 17.293 ms stddev 12.830, 0 failed
progress: 12.0 s, 428.6 tps, lat 18.632 ms stddev 13.670, 0 failed
progress: 18.0 s, 479.6 tps, lat 16.640 ms stddev 11.468, 0 failed
progress: 24.0 s, 468.3 tps, lat 17.074 ms stddev 11.071, 0 failed
progress: 30.0 s, 543.2 tps, lat 14.689 ms stddev 10.260, 0 failed
progress: 36.0 s, 567.2 tps, lat 14.093 ms stddev 9.284, 0 failed
progress: 42.0 s, 574.0 tps, lat 13.894 ms stddev 10.096, 0 failed
progress: 48.0 s, 563.0 tps, lat 14.179 ms stddev 10.559, 0 failed
progress: 54.0 s, 579.7 tps, lat 13.768 ms stddev 9.521, 0 failed
progress: 60.0 s, 562.8 tps, lat 14.188 ms stddev 10.142, 0 failed
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 8
number of threads: 1
maximum number of tries: 1
duration: 60 s
number of transactions actually processed: 31327
number of failed transactions: 0 (0.000%)
latency average = 15.271 ms
latency stddev = 10.958 ms
initial connection time = 98.863 ms
tps = 522.765192 (without initial connection time)
```
2 раз, получше, но не сильно
```
daa@daa-VMware-Virtual-Platform:~$ pgbench -c 8 -P 6 -T 60 -U postgres demo
pgbench (15.8 (Ubuntu 15.8-1.pgdg24.04+1))
starting vacuum...end.
progress: 6.0 s, 516.3 tps, lat 15.217 ms stddev 10.346, 0 failed
progress: 12.0 s, 589.3 tps, lat 13.544 ms stddev 8.845, 0 failed
progress: 18.0 s, 534.9 tps, lat 14.934 ms stddev 9.631, 0 failed
progress: 24.0 s, 546.8 tps, lat 14.611 ms stddev 9.101, 0 failed
progress: 30.0 s, 543.2 tps, lat 14.694 ms stddev 9.407, 0 failed
progress: 36.0 s, 566.2 tps, lat 14.067 ms stddev 9.192, 0 failed
progress: 42.0 s, 502.5 tps, lat 15.941 ms stddev 13.496, 0 failed
progress: 48.0 s, 555.3 tps, lat 14.383 ms stddev 9.177, 0 failed
progress: 54.0 s, 567.5 tps, lat 14.069 ms stddev 9.275, 0 failed
progress: 60.0 s, 545.0 tps, lat 14.662 ms stddev 9.820, 0 failed
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 8
number of threads: 1
maximum number of tries: 1
duration: 60 s
number of transactions actually processed: 32810
number of failed transactions: 0 (0.000%)
latency average = 14.586 ms
latency stddev = 9.883 ms
initial connection time = 90.240 ms
tps = 547.416551 (without initial connection time)
```

#### Проверим текущую конфигурацию, настройки по дефолту

```
daa@daa-VMware-Virtual-Platform:~$ pg_conftool show all
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

#### Теперь попробуем изменить конфигурацию в соответствии с ДЗ. Внесем необходимые изменения в postgresql.conf

```
max_connections = 40 # максимальное число подключиений к БД
shared_buffers = 1GB # размер буфферного кэша под страницы, рекомендуемое значение от ОЗУ 25%, соответствует рекомендации.
effective_cache_size = 3GB # эффиктивный размер дискового кеша для планировщика. По документации по умолчанию 4GB. Это оценочный параметр, не влияет на размеры выделяемой памяти.
maintenance_work_mem = 512MB # Задаёт максимальный объём памяти, который будет использовать каждый рабочий процесс автоочистки. Максимум используется 1Gb, даже если указать большее значение
checkpoint_completion_target = 0.9 # Задаёт целевое время для завершения процедуры контрольной точки, как долю общего времени между контрольными точками. Значение по умолчанию — 0.9 
wal_buffers = 16MB # Объём разделяемой памяти, который будет использоваться для буферизации данных WAL, ещё не записанных на диск. По умолчанию 1/32 от общего размера памяти, но не меньше чем 64 КБ и не больше чем размер одного сегмента WAL (обычно 16 МБ). У нас максимальное значение
default_statistics_target = 500 # Чем больше установленное значение, тем больше времени требуется для выполнения ANALYZE, но тем выше может быть качество оценок планировщика. По умлочанию - 100
random_page_cost = 4.0 # Задаёт приблизительную стоимость чтения одной произвольной страницы с диска. Значение по умолчанию равно 4.0. 
effective_io_concurrency = 2 #  Задаёт допустимое число параллельных операций ввода/вывода, которое говорит Postgres Pro о том, сколько операций ввода/вывода могут быть выполнены одновременно. Диски SSD и другие виды хранилища в памяти часто могут обрабатывать множество параллельных запросов, так что оптимальным числом может быть несколько сотен.
work_mem = 6553kB # Задаёт базовый максимальный объём памяти, который будет использоваться во внутренних операциях при обработке запросов, прежде чем будут задействованы временные файлы на диске. Значение по умолчанию — четыре мегабайта (4MB), у нас немного больше. 
min_wal_size = 4GB # Пока WAL занимает на диске меньше этого объёма, старые файлы WAL в контрольных точках всегда перерабатываются, а не удаляются. Значение по умолчанию — 80 МБ.
max_wal_size = 16GB # Максимальный размер, до которого может вырастать WAL во время автоматических контрольных точек. Значение по умолчанию — 1 ГБ. Увеличение этого параметра может привести к увеличению времени, которое потребуется для восстановления после сбоя. Странный выбор, учитывая 10Gb диск.
```

![image](https://github.com/user-attachments/assets/21766e12-ad81-426e-88ef-af21ed629fbf)

#### Перезапустим кластер и проверим конфигурацию

```
daa@daa-VMware-Virtual-Platform:~$ pg_conftool show all
checkpoint_completion_target = 0.9
cluster_name = '15/main'
data_directory = '/var/lib/postgresql/15/main'
datestyle = 'iso, dmy'
default_statistics_target = 500
default_text_search_config = 'pg_catalog.russian'
dynamic_shared_memory_type = posix
effective_cache_size = 3GB
effective_io_concurrency = 2
external_pid_file = '/var/run/postgresql/15-main.pid'
hba_file = '/etc/postgresql/15/main/pg_hba.conf'
ident_file = '/etc/postgresql/15/main/pg_ident.conf'
lc_messages = 'ru_RU.UTF-8'
lc_monetary = 'ru_RU.UTF-8'
lc_numeric = 'ru_RU.UTF-8'
lc_time = 'ru_RU.UTF-8'
log_line_prefix = '%m [%p] %q%u@%d '
log_timezone = 'Europe/Moscow'
maintenance_work_mem = 512MB
max_connections = 40
max_wal_size = 16GB
min_wal_size = 4GB
port = 5432
random_page_cost = 4.0
shared_buffers = 1GB
ssl = on
ssl_cert_file = '/etc/ssl/certs/ssl-cert-snakeoil.pem'
ssl_key_file = '/etc/ssl/private/ssl-cert-snakeoil.key'
timezone = 'Europe/Moscow'
unix_socket_directories = '/var/run/postgresql'
wal_buffers = 16MB
work_mem = 6553kB
```

#### Теперь pgbench -c8 -P 6 -T 60 -U postgres postgres. Но выполним операцию 2 раза

1 раз
```
daa@daa-VMware-Virtual-Platform:~$ pgbench -c 8 -P 6 -T 60 -U postgres demo
pgbench (15.8 (Ubuntu 15.8-1.pgdg24.04+1))
starting vacuum...end.
progress: 6.0 s, 453.6 tps, lat 17.293 ms stddev 12.830, 0 failed
progress: 12.0 s, 428.6 tps, lat 18.632 ms stddev 13.670, 0 failed
progress: 18.0 s, 479.6 tps, lat 16.640 ms stddev 11.468, 0 failed
progress: 24.0 s, 468.3 tps, lat 17.074 ms stddev 11.071, 0 failed
progress: 30.0 s, 543.2 tps, lat 14.689 ms stddev 10.260, 0 failed
progress: 36.0 s, 567.2 tps, lat 14.093 ms stddev 9.284, 0 failed
progress: 42.0 s, 574.0 tps, lat 13.894 ms stddev 10.096, 0 failed
progress: 48.0 s, 563.0 tps, lat 14.179 ms stddev 10.559, 0 failed
progress: 54.0 s, 579.7 tps, lat 13.768 ms stddev 9.521, 0 failed
progress: 60.0 s, 562.8 tps, lat 14.188 ms stddev 10.142, 0 failed
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 8
number of threads: 1
maximum number of tries: 1
duration: 60 s
number of transactions actually processed: 31327
number of failed transactions: 0 (0.000%)
latency average = 15.271 ms
latency stddev = 10.958 ms
initial connection time = 98.863 ms
tps = 522.765192 (without initial connection time)
```

2 раз

```
daa@daa-VMware-Virtual-Platform:~$ pgbench -c 8 -P 6 -T 60 -U postgres demo
pgbench (15.8 (Ubuntu 15.8-1.pgdg24.04+1))
starting vacuum...end.
progress: 6.0 s, 516.3 tps, lat 15.217 ms stddev 10.346, 0 failed
progress: 12.0 s, 589.3 tps, lat 13.544 ms stddev 8.845, 0 failed
progress: 18.0 s, 534.9 tps, lat 14.934 ms stddev 9.631, 0 failed
progress: 24.0 s, 546.8 tps, lat 14.611 ms stddev 9.101, 0 failed
progress: 30.0 s, 543.2 tps, lat 14.694 ms stddev 9.407, 0 failed
progress: 36.0 s, 566.2 tps, lat 14.067 ms stddev 9.192, 0 failed
progress: 42.0 s, 502.5 tps, lat 15.941 ms stddev 13.496, 0 failed
progress: 48.0 s, 555.3 tps, lat 14.383 ms stddev 9.177, 0 failed
progress: 54.0 s, 567.5 tps, lat 14.069 ms stddev 9.275, 0 failed
progress: 60.0 s, 545.0 tps, lat 14.662 ms stddev 9.820, 0 failed
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 8
number of threads: 1
maximum number of tries: 1
duration: 60 s
number of transactions actually processed: 32810
number of failed transactions: 0 (0.000%)
latency average = 14.586 ms
latency stddev = 9.883 ms
initial connection time = 90.240 ms
tps = 547.416551 (without initial connection time)
```

#### Сравним результаты. Немного лучше, но не существенно.

До изменений

```
scaling factor: 1
query mode: simple
number of clients: 8
number of threads: 1
maximum number of tries: 1
duration: 60 s
number of transactions actually processed: 31327
number of failed transactions: 0 (0.000%)
latency average = 15.271 ms
latency stddev = 10.958 ms
initial connection time = 98.863 ms
tps = 522.765192 (without initial connection time)
```

После измений

```
scaling factor: 1
query mode: simple
number of clients: 8
number of threads: 1
maximum number of tries: 1
duration: 60 s
number of transactions actually processed: 32810
number of failed transactions: 0 (0.000%)
latency average = 14.586 ms
latency stddev = 9.883 ms
initial connection time = 90.240 ms
tps = 547.416551 (without initial connection time)
```

#### Теперь попробуем добавить nvme диск и перенесем на него БД, подробно расписывать не буду, алгоритм похож на тот, что мы делали при выполнении ДЗ 3.
#### Сохраним конфигурацию с изменениями, по пути данных видно, что данные находятся в другой директории.

```
nvme0n1     259:0    0    10G  0 disk
└─nvme0n1p1 259:1    0    10G  0 part /mnt/data
```
```
daa@daa-VMware-Virtual-Platform:/etc/postgresql/15/main$ pg_conftool show all
checkpoint_completion_target = 0.9
cluster_name = '15/main'
data_directory = '/mnt/data/15/main'
datestyle = 'iso, dmy'
default_statistics_target = 500
default_text_search_config = 'pg_catalog.russian'
dynamic_shared_memory_type = posix
effective_cache_size = 3GB
effective_io_concurrency = 2
external_pid_file = '/var/run/postgresql/15-main.pid'
hba_file = '/etc/postgresql/15/main/pg_hba.conf'
ident_file = '/etc/postgresql/15/main/pg_ident.conf'
lc_messages = 'ru_RU.UTF-8'
lc_monetary = 'ru_RU.UTF-8'
lc_numeric = 'ru_RU.UTF-8'
lc_time = 'ru_RU.UTF-8'
log_line_prefix = '%m [%p] %q%u@%d '
log_timezone = 'Europe/Moscow'
maintenance_work_mem = 512MB
max_connections = 40
max_wal_size = 16GB
min_wal_size = 4GB
port = 5432
random_page_cost = 4.0
shared_buffers = 1GB
ssl = on
ssl_cert_file = '/etc/ssl/certs/ssl-cert-snakeoil.pem'
ssl_key_file = '/etc/ssl/private/ssl-cert-snakeoil.key'
timezone = 'Europe/Moscow'
unix_socket_directories = '/var/run/postgresql'
wal_buffers = 16MB
work_mem = 6553kB
```

#### Теперь pgbench -c8 -P 6 -T 60 -U postgres postgres. Но выполним операцию 2 раза

1 раз

```
daa@daa-VMware-Virtual-Platform:/etc/postgresql/15/main$ pgbench -c 8 -P 6 -T 60                                                          -U postgres demo
pgbench (15.8 (Ubuntu 15.8-1.pgdg24.04+1))
starting vacuum...end.
progress: 6.0 s, 517.3 tps, lat 15.192 ms stddev 9.661, 0 failed
progress: 12.0 s, 562.0 tps, lat 14.200 ms stddev 9.047, 0 failed
progress: 18.0 s, 557.3 tps, lat 14.328 ms stddev 9.233, 0 failed
progress: 24.0 s, 555.5 tps, lat 14.373 ms stddev 9.099, 0 failed
progress: 30.0 s, 549.1 tps, lat 14.545 ms stddev 9.721, 0 failed
progress: 36.0 s, 554.0 tps, lat 14.412 ms stddev 9.817, 0 failed
progress: 42.0 s, 532.8 tps, lat 14.980 ms stddev 10.602, 0 failed
progress: 48.0 s, 533.7 tps, lat 14.962 ms stddev 9.625, 0 failed
progress: 54.0 s, 575.8 tps, lat 13.862 ms stddev 8.585, 0 failed
progress: 60.0 s, 563.7 tps, lat 14.157 ms stddev 9.141, 0 failed
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 8
number of threads: 1
maximum number of tries: 1
duration: 60 s
number of transactions actually processed: 33016
number of failed transactions: 0 (0.000%)
latency average = 14.491 ms
latency stddev = 9.467 ms
initial connection time = 82.682 ms
tps = 550.764512 (without initial connection time)
```

2 раз

```
daa@daa-VMware-Virtual-Platform:/etc/postgresql/15/main$ pgbench -c 8 -P 6 -T 60                                                          -U postgres demo
pgbench (15.8 (Ubuntu 15.8-1.pgdg24.04+1))
starting vacuum...end.
progress: 6.0 s, 538.6 tps, lat 14.592 ms stddev 9.290, 0 failed
progress: 12.0 s, 551.0 tps, lat 14.474 ms stddev 9.716, 0 failed
progress: 18.0 s, 527.0 tps, lat 15.163 ms stddev 10.118, 0 failed
progress: 24.0 s, 513.2 tps, lat 15.551 ms stddev 12.228, 0 failed
progress: 30.0 s, 563.8 tps, lat 14.152 ms stddev 9.221, 0 failed
progress: 36.0 s, 551.3 tps, lat 14.472 ms stddev 9.510, 0 failed
progress: 42.0 s, 560.5 tps, lat 14.247 ms stddev 8.912, 0 failed
progress: 48.0 s, 571.0 tps, lat 13.977 ms stddev 9.139, 0 failed
progress: 54.0 s, 565.0 tps, lat 14.139 ms stddev 9.420, 0 failed
progress: 60.0 s, 567.0 tps, lat 14.080 ms stddev 9.155, 0 failed
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 8
number of threads: 1
maximum number of tries: 1
duration: 60 s
number of transactions actually processed: 33059
number of failed transactions: 0 (0.000%)
latency average = 14.470 ms
latency stddev = 9.698 ms
initial connection time = 83.944 ms
tps = 551.609447 (without initial connection time)
```

#### Сравним результаты. Видимых изменений нет, скорее погрешность. Отдельное добавление ssd с учетом использования SSD диска на хосте виртуализации ожидаемо не дало результата)

До изменений

```
scaling factor: 1
query mode: simple
number of clients: 8
number of threads: 1
maximum number of tries: 1
duration: 60 s
number of transactions actually processed: 32810
number of failed transactions: 0 (0.000%)
latency average = 14.586 ms
latency stddev = 9.883 ms
initial connection time = 90.240 ms
tps = 547.416551 (without initial connection time)
```

После измений

```
scaling factor: 1
query mode: simple
number of clients: 8
number of threads: 1
maximum number of tries: 1
duration: 60 s
number of transactions actually processed: 33059
number of failed transactions: 0 (0.000%)
latency average = 14.470 ms
latency stddev = 9.698 ms
initial connection time = 83.944 ms
tps = 551.609447 (without initial connection time)
```

#### Выводы по ситуации писла на следующий день и при повторном тестировании обнаружил результат, объяснить который у меня пока не получается.
#### Наблюдается явное улучшение производительности, при этом кластер не перезапускался.

```
daa@daa-VMware-Virtual-Platform:~$ pgbench -c 8 -P 6 -T 60 -U postgres demo
pgbench (15.8 (Ubuntu 15.8-1.pgdg24.04+1))
starting vacuum...end.
progress: 6.0 s, 1916.3 tps, lat 4.142 ms stddev 3.870, 0 failed
progress: 12.0 s, 2367.9 tps, lat 3.374 ms stddev 2.674, 0 failed
progress: 18.0 s, 2291.3 tps, lat 3.486 ms stddev 2.679, 0 failed
progress: 24.0 s, 2193.0 tps, lat 3.638 ms stddev 2.805, 0 failed
progress: 30.0 s, 2212.6 tps, lat 3.599 ms stddev 2.793, 0 failed
progress: 36.0 s, 2200.8 tps, lat 3.643 ms stddev 2.819, 0 failed
progress: 42.0 s, 2210.2 tps, lat 3.614 ms stddev 2.847, 0 failed
progress: 48.0 s, 2231.4 tps, lat 3.582 ms stddev 2.903, 0 failed
progress: 54.0 s, 2294.9 tps, lat 3.480 ms stddev 2.907, 0 failed
progress: 60.0 s, 2159.0 tps, lat 3.700 ms stddev 6.926, 0 failed
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 8
number of threads: 1
maximum number of tries: 1
duration: 60 s
number of transactions actually processed: 132471
number of failed transactions: 0 (0.000%)
latency average = 3.616 ms
latency stddev = 3.527 ms
initial connection time = 37.278 ms
tps = 2208.683741 (without initial connection time)
```

```
daa@daa-VMware-Virtual-Platform:~$ iostat nvme0n1 5

avg-cpu:  %user   %nice %system %iowait  %steal   %idle
           8,16    0,00   37,26    3,98    0,00   50,60

Device             tps    kB_read/s    kB_wrtn/s    kB_dscd/s    kB_read    kB_wrtn    kB_dscd
nvme0n1        2367,40         0,00     20095,20         0,00          0     100476          0
```

#### Но я рискну предположить, что такое ограничение с производительностью БД все таки связано с производительностью диска. Но не понятно, почему она изменилась.

#### Создадим таблицу с текстовым полем и заполнить случайными или сгенерированными данным в размере 1млн строк

```
daa@daa-VMware-Virtual-Platform:~$ sudo -u postgres psql
[sudo] пароль для daa:
psql (17.0 (Ubuntu 17.0-1.pgdg24.04+1), сервер 15.8 (Ubuntu 15.8-1.pgdg24.04+1))
Введите "help", чтобы получить справку.

postgres=# CREATE TABLE text_table (
    id SERIAL PRIMARY KEY,
    text TEXT
);
CREATE TABLE
postgres=# INSERT INTO text_table
SELECT id, random()::TEXT
FROM generate_series(1, 1000000) AS id;
INSERT 0 1000000
```

#### Посмотрим размер файла

```
postgres=# SELECT pg_size_pretty(pg_total_relation_size('text_table'));
 pg_size_pretty
----------------
 72 MB
(1 строка)
```

#### 5 раз обновим все строчки и добавить к каждой строчке любой символ

```
postgres=# update text_table SET text = random()::TEXT || 'Q';
UPDATE 1000000
postgres=# update text_table SET text = random()::TEXT || '!';
UPDATE 1000000
postgres=# update text_table SET text = random()::TEXT || 'й';
UPDATE 1000000
postgres=# update text_table SET text = random()::TEXT || '5';
UPDATE 1000000
postgres=# update text_table SET text = random()::TEXT || 'Йw!2';
UPDATE 1000000
```

#### Посмотрим количество мертвых строчек в таблице и когда последний раз приходил автовакуум. По всей видимости автовакуум уже прошел

```
postgres=# SELECT relname, n_live_tup, n_dead_tup, trunc(100*n_dead_tup/(n_live_tup+1))::float "ratio%", last_autovacuum FROM pg_stat_user_tables WHERE relname = 'text_table';
  relname   | n_live_tup | n_dead_tup | ratio% |        last_autovacuum
------------+------------+------------+--------+-------------------------------
 text_table |    1000000 |          0 |      0 | 2024-10-18 12:56:52.362114+03
(1 строка)
```

#### Повторим предыдущий шаг и проверим еще раз. Теперь я успел)

```
postgres=# SELECT relname, n_live_tup, n_dead_tup, trunc(100*n_dead_tup/(n_live_tup+1))::float "ratio%", last_autovacuum FROM pg_stat_user_tables WHERE relname = 'text_table';
  relname   | n_live_tup | n_dead_tup | ratio% |        last_autovacuum
------------+------------+------------+--------+-------------------------------
 text_table |    1000000 |    4991055 |    499 | 2024-10-18 12:56:52.362114+03
(1 строка)
```
```
postgres=# SELECT relname, n_live_tup, n_dead_tup, trunc(100*n_dead_tup/(n_live_tup+1))::float "ratio%", last_autovacuum FROM pg_stat_user_tables WHERE relname = 'text_table';
  relname   | n_live_tup | n_dead_tup | ratio% |        last_autovacuum
------------+------------+------------+--------+-------------------------------
 text_table |    1000000 |          0 |      0 | 2024-10-18 12:59:53.629429+03
(1 строка)
```

#### 5 раз обновить все строчки и добавить к каждой строчке любой символ. По скольку мы это уже сделали посмотрим на размер

```
postgres=# SELECT pg_size_pretty(pg_total_relation_size('text_table'));          pg_size_pretty
----------------
 391 MB
(1 строка)
```

#### Отключим автовакуум на нашей таблице

```
postgres=# alter table text_table set (autovacuum_enabled=off);
ALTER TABLE
```

#### 10 раз обновим все строчки и добавить к каждой строчке любой символ.

```
postgres=# update text_table SET text = random()::TEXT || '5';
UPDATE 1000000
```

#### Посмотрим размер файла с таблицей

```
postgres=# SELECT pg_size_pretty(pg_total_relation_size('text_table'));
 pg_size_pretty
----------------
 640 MB
(1 строка)
```

#### Увеличение объема связано с тем, что мы добавили данные, но т.к. автовакуум отключен, мертвые строки в таблице все равно присутствуют.

```
postgres=# SELECT relname, n_live_tup, n_dead_tup, trunc(100*n_dead_tup/(n_live_tup+1))::float "ratio%", last_autovacuum FROM pg_stat_user_tables WHERE relname = 'text_table';
  relname   | n_live_tup | n_dead_tup | ratio% |        last_autovacuum
------------+------------+------------+--------+-------------------------------
 text_table |    1000000 |    9968050 |    996 | 2024-10-18 12:59:53.629429+03
(1 строка)
```

#### Если включить автовакуум, то объем не изменится, но мертвые строки будут почищены

```
postgres=# alter table text_table set (autovacuum_enabled=on);
postgres=# SELECT relname, n_live_tup, n_dead_tup, trunc(100*n_dead_tup/(n_live_tup+1))::float "ratio%", last_autovacuum FROM pg_stat_user_tables WHERE relname = 'text_table';
  relname   | n_live_tup | n_dead_tup | ratio% |       last_autovacuum
------------+------------+------------+--------+------------------------------
 text_table |    1000000 |          0 |      0 | 2024-10-18 13:10:18.10202+03
(1 строка)

postgres=# SELECT pg_size_pretty(pg_total_relation_size('text_table'));          pg_size_pretty
----------------
 640 MB
(1 строка)
```

#### Задание со *:
#### Написать анонимную процедуру, в которой в цикле 10 раз обновятся все строчки в искомой таблице.
#### Не забыть вывести номер шага цикла.

Процедура выполняется в блоке DO
Анонимные блоки кода заключаются в $$

```
postgres=# DO $$
BEGIN
    FOR i IN 1..10 LOOP
        update text_table SET text = gen_random_uuid() || 'Q';
        RAISE NOTICE 'Шаг №:%', i;
    END LOOP;
END;
$$ LANGUAGE plpgsql;
ЗАМЕЧАНИЕ:  Шаг №:1
ЗАМЕЧАНИЕ:  Шаг №:2
ЗАМЕЧАНИЕ:  Шаг №:3
ЗАМЕЧАНИЕ:  Шаг №:4
ЗАМЕЧАНИЕ:  Шаг №:5
ЗАМЕЧАНИЕ:  Шаг №:6
ЗАМЕЧАНИЕ:  Шаг №:7
ЗАМЕЧАНИЕ:  Шаг №:8
ЗАМЕЧАНИЕ:  Шаг №:9
ЗАМЕЧАНИЕ:  Шаг №:10
DO
postgres=# SELECT pg_size_pretty(pg_total_relation_size('text_table'));
 pg_size_pretty
----------------
 912 MB
(1 строка)
```

