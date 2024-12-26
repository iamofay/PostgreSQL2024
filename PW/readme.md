### Дунай А.А. Проектная работа. Создание и тестирование высоконагруженного отказоустойчивого кластера PostgreSQL на базе Patroni.

Для начала определимся с перечнем компонентов системы и построим схему.
В своей реализации я буду использовать 7 ВМ на базе Ubuntu:

- etcd1 192.168.1.106
- etcd2 192.168.1.109
- etcd3 192.168.1.110
- patroni1 192.168.1.111
- patroni2 192.168.1.112 
- haproxy1 192.168.1.113
- haproxy1 192.168.1.114

На которых будут рзмещены компоненты:

- etcd
- patroni
- pgbouncer
- haproxy
- keepalive

![otus postgresql](https://github.com/user-attachments/assets/d0d9a92e-f827-47c1-981c-5b2c0145bb09)

#### 1. Начнем с подготовки кластера etcd

- etcd1 192.168.1.106
- etcd2 192.168.1.109
- etcd3 192.168.1.110

etcd — база данных для кластера patroni, критически важный компонент любого кластера. Он хранит в себе всю информацию, нужную для его стабильной работы. Основная задача — обеспечить консистентность данных и отказоустойчивость кластера. Если etcd запущен в нескольких репликах, будь то на отдельных машинах или машинах с Kubernetes, это обеспечит непрерывность работы кластера Kubernetes в случае сбоя отдельных нод etcd.

![image](https://github.com/user-attachments/assets/02174db7-6a00-40d2-9936-710762d7f82a)
![image](https://github.com/user-attachments/assets/b3339d60-a147-487c-adaf-e53ca34ee0ce)

1.1. Установим etcd
  
Используем репозиторий на github, установим версию v3.5.17

[https://github.com/etcd-io/etcd/releases/tag/v3.5.17](https://github.com/etcd-io/etcd/releases/tag/v3.5.17) 

Результаты установки:

```
GOOGLE_URL=https://storage.googleapis.com/etcd
GITHUB_URL=https://github.com/etcd-io/etcd/releases/download
DOWNLOAD_URL=${GOOGLE_URL}
```

```
daa@etcd1:~$ curl -L ${DOWNLOAD_URL}/${ETCD_VER}/etcd-${ETCD_VER}-linux-amd64.tar.gz -o /tmp/etcd-${ETCD_VER}-linux-amd64.tar.gz
tar xzvf /tmp/etcd-${ETCD_VER}-linux-amd64.tar.gz -C /tmp/etcd-download-test --strip-components=1
rm -f /tmp/etcd-${ETCD_VER}-linux-amd64.tar.gz
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100 19.5M  100 19.5M    0     0  7497k      0  0:00:02  0:00:02 --:--:-- 7496k
etcd-v3.5.17-linux-amd64/etcd
etcd-v3.5.17-linux-amd64/etcdctl
etcd-v3.5.17-linux-amd64/etcdutl
etcd-v3.5.17-linux-amd64/README.md
etcd-v3.5.17-linux-amd64/README-etcdctl.md
etcd-v3.5.17-linux-amd64/READMEv2-etcdctl.md
etcd-v3.5.17-linux-amd64/README-etcdutl.md
etcd-v3.5.17-linux-amd64/Documentation/
etcd-v3.5.17-linux-amd64/Documentation/README.md
etcd-v3.5.17-linux-amd64/Documentation/dev-guide/
etcd-v3.5.17-linux-amd64/Documentation/dev-guide/apispec/
etcd-v3.5.17-linux-amd64/Documentation/dev-guide/apispec/swagger/
etcd-v3.5.17-linux-amd64/Documentation/dev-guide/apispec/swagger/rpc.swagger.json
etcd-v3.5.17-linux-amd64/Documentation/dev-guide/apispec/swagger/v3election.swagger.json
etcd-v3.5.17-linux-amd64/Documentation/dev-guide/apispec/swagger/v3lock.swagger.json
daa@etcd1:~$ /tmp/etcd-download-test/etcd --version
/tmp/etcd-download-test/etcdctl version
/tmp/etcd-download-test/etcdutl version
etcd Version: 3.5.17
Git SHA: 507c0de
Go Version: go1.22.9
Go OS/Arch: linux/amd64
etcdctl version: 3.5.17
API version: 3.5
etcdutl version: 3.5.17
API version: 3.5

daa@etcd1:~$ sudo nano /etc/systemd/system/etcd.service
daa@etcd1:~$ etcd --version
etcd Version: 3.5.17
Git SHA: 507c0de
Go Version: go1.22.9
Go OS/Arch: linux/amd64
```

1.2. Добавим в PATH:

```
sudo cp /tmp/etcd-download-test/etcd* /usr/local/bin
```

Теперь создадим свой служебный файл:

```
sudo vim /etc/systemd/system/etcd.service
```

И добавим в него следующую конфигурацию:

```
[Unit]
Description=etcd

[Service]
Type=notify
EnvironmentFile=/etc/default/etcd
ExecStart=/usr/local/bin/etcd
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
```

1.3. Настроим кластер:

Откройте конфигурационный файл для редактирования:

```
sudo nano /etc/default/etcd
```

Этот файл содержит все переменные среды для кластера. Нам нужно определить все переменные на всех трех узлах по аналогии:

```
  GNU nano 7.2                                            /etc/default/etcd
ETCD_NAME=etcd1
ETCD_DATA_DIR=/var/lib/etcd/default
ETCD_INITIAL_CLUSTER=etcd1=http://192.168.1.106:2380,etcd2=http://192.168.1.109:2380,etcd3=http://192.168.1.110:2380
ETCD_INITIAL_ADVERTISE_PEER_URLS=http://192.168.1.106:2380
ETCD_ADVERTISE_CLIENT_URLS=http://192.168.1.106:2379
ETCD_LISTEN_PEER_URLS=http://192.168.1.106:2380
ETCD_LISTEN_CLIENT_URLS=http://192.168.1.106:2379,http://localhost:2379
ETCD_INITIAL_CLUSTER_TOKEN="devops_token"
ETCD_INITIAL_CLUSTER_STATE="new"
ETCD_ENABLE_V2="true"
ETCD_HEARTBEAT_INTERVAL="1000"
ETCD_ELECTION_TIMEOUT="5000"
```

Рассмотрим введённые параметры:

   - ETCD_NAME — имя этого узла кластера. Должно быть уникально в кластере;
   - ETCD_LISTEN_CLIENT_URLS — точка подключения для клиентов кластера;
   - ETCD_ADVERTISE_CLIENT_URLS — список URL-адресов, по которым его могут найти остальные узлы кластера;
   - ETCD_LISTEN_PEER_URLS — точка подключения для остальных узлов кластера;
   - ETCD_INITIAL_ADVERTISE_PEER_URLS — начальный список URL-адресов, по которым его могут найти остальные узлы кластера;
   - ETCD_INITIAL_CLUSTER_TOKEN — токен кластера, должен совпадать на всех узлах кластера;
   - ETCD_INITIAL_CLUSTER — список узлов кластера на момент запуска;
   - ETCD_INITIAL_CLUSTER_STATE — может принимать два значения: new и existing;
   - ETCD_DATA_DIR — расположение каталога данных кластера;
   - ETCD_ELECTION_TIMEOUT — время в миллисекундах, которое проходит между последним принятым оповещением от лидера кластера, до попытки захватить роль лидера на ведомом узле;
   - ETCD_HEARTBEAT_INTERVAL — время в миллисекундах, между рассылками лидером оповещений о том, что он всё ещё лидер.
   - ETCD_CERT_FILE — путь до файла сертификата сервера;
   - ETCD_KEY_FILE — путь до файла закрытого ключа;
   - ETCD_TRUSTED_CA_FILE — путь до файла корневого CA;
   - ETCD_CLIENT_CERT_AUTH — может принимать два значения: true и false;
   - ETCD_PEER_CERT_FILE — путь до файла сертификата сервера;
   - ETCD_PEER_KEY_FILE — путь до файла закрытого ключа;
   - ETCD_PEER_TRUSTED_CA_FILE — путь до файла корневого CA;
   - ETCD_PEER_CLIENT_CERT_AUTH — может принимать два значения: true и false;

После внесения изменений на всех трех узлах запустим службу etcd с помощью команд:

```
sudo systemctl daemon-reload
sudo systemctl enable --now etcd
```
Или рестарт

```
sudo systemctl restart etcd
```

1.4. Убедимся, что служба работает на всех узлах:

```
daa@etcd1:~$ systemctl status etcd
● etcd.service - etcd
     Loaded: loaded (/etc/systemd/system/etcd.service; enabled; preset: enabled)
     Active: active (running) since Tue 2024-12-17 13:39:07 MSK; 12s ago
   Main PID: 12308 (etcd)
      Tasks: 7 (limit: 4558)
     Memory: 17.4M (peak: 17.9M)
        CPU: 872ms
     CGroup: /system.slice/etcd.service
             └─12308 /usr/local/bin/etcd

Dec 17 13:39:09 etcd1 etcd[12308]: {"level":"info","ts":"2024-12-17T13:39:09.091520+0300","caller":"rafthttp/stream.go:249","msg>
Dec 17 13:39:09 etcd1 etcd[12308]: {"level":"info","ts":"2024-12-17T13:39:09.092123+0300","caller":"rafthttp/peer_status.go:53",>
Dec 17 13:39:09 etcd1 etcd[12308]: {"level":"info","ts":"2024-12-17T13:39:09.092058+0300","caller":"rafthttp/stream.go:249","msg>
Dec 17 13:39:09 etcd1 etcd[12308]: {"level":"info","ts":"2024-12-17T13:39:09.092157+0300","caller":"rafthttp/stream.go:274","msg>
Dec 17 13:39:09 etcd1 etcd[12308]: {"level":"info","ts":"2024-12-17T13:39:09.092167+0300","caller":"rafthttp/stream.go:274","msg>
Dec 17 13:39:09 etcd1 etcd[12308]: {"level":"info","ts":"2024-12-17T13:39:09.128262+0300","caller":"rafthttp/stream.go:412","msg>
Dec 17 13:39:09 etcd1 etcd[12308]: {"level":"info","ts":"2024-12-17T13:39:09.128265+0300","caller":"rafthttp/stream.go:412","msg>
Dec 17 13:39:11 etcd1 etcd[12308]: {"level":"info","ts":"2024-12-17T13:39:11.723260+0300","caller":"etcdserver/server.go:2656",">
Dec 17 13:39:11 etcd1 etcd[12308]: {"level":"info","ts":"2024-12-17T13:39:11.723985+0300","caller":"membership/cluster.go:576",">
Dec 17 13:39:11 etcd1 etcd[12308]: {"level":"info","ts":"2024-12-17T13:39:11.724083+0300","caller":"etcdserver/server.go:2675",">
lines 1-20/20 (END)
```

На данный момент кластер запущен и работает. Чтобы убедиться в этом, используйте приведенную ниже команду и замените IP-адрес на IP-адрес любого сервера etcd.

```
daa@etcd1:~$ sudo etcdctl member list
2125f1467958dff8, started, etcd1, http://192.168.1.106:2380, http://192.168.1.106:2379, false
265b9c61ffe85ba1, started, etcd3, http://192.168.1.110:2380, http://192.168.1.110:2379, false
97ba8852ebaddf74, started, etcd2, http://192.168.1.109:2380, http://192.168.1.109:2379, false
daa@etcd1:~$ etcdctl -w table endpoint --cluster status
+---------------------------+------------------+---------+---------+-----------+------------+-----------+------------+--------------------+--------+
|         ENDPOINT          |        ID        | VERSION | DB SIZE | IS LEADER | IS LEARNER | RAFT TERM | RAFT INDEX | RAFT APPLIED INDEX | ERRORS |
+---------------------------+------------------+---------+---------+-----------+------------+-----------+------------+--------------------+--------+
| http://192.168.1.106:2379 | 2125f1467958dff8 |  3.5.17 |   20 kB |     false |      false |         3 |         10 |                 10 |        |
| http://192.168.1.110:2379 | 265b9c61ffe85ba1 |  3.5.17 |   20 kB |      true |      false |         3 |         10 |                 10 |        |
| http://192.168.1.109:2379 | 97ba8852ebaddf74 |  3.5.17 |   20 kB |     false |      false |         3 |         10 |                 10 |        |
+---------------------------+------------------+---------+---------+-----------+------------+-----------+------------+--------------------+--------+
```

#### 2. Теперь приступим к развертыванию Postgresql+patroni

- patroni1 192.168.1.111
- patroni2 192.168.1.112

Основной задачей Patroni является обеспечение надежного переключения роли ведущего узла на резервный узел. Для максимальной доступности СУБД в кластере Patroni необходимо хранить и при необходимости изменять информацию о роли узлов. Для этого используется DCS, которое реализуется с помощью etcd или consul. DCS представляет собой распределенное хранилище конфигурации ключ/значение. В нашем случае кластер ETCD.

2.1. Установим postgre (информация взята из примеров ДЗ, в проектной работе выполнено по аналогии)

Утсановим некоторые пакеты

```
daa@daa-VMware-Virtual-Platform:~$ sudo apt install dirmngr ca-certificates software-properties-common apt-transport-https lsb-release curl -y
Чтение списков пакетов… Готово
Построение дерева зависимостей… Готово
Чтение информации о состоянии… Готово
Уже установлен пакет dirmngr самой новой версии (2.4.4-2ubuntu17).
dirmngr помечен как установленный вручную.
Уже установлен пакет ca-certificates самой новой версии (20240203).
Уже установлен пакет software-properties-common самой новой версии (0.99.48).
software-properties-common помечен как установленный вручную.
Уже установлен пакет lsb-release самой новой версии (12.0-2).
lsb-release помечен как установленный вручную.
Уже установлен пакет curl самой новой версии (8.5.0-2ubuntu10.4).
Следующие НОВЫЕ пакеты будут установлены:
  apt-transport-https
Обновлено 0 пакетов, установлено 1 новых пакетов, для удаления отмечено 0 пакетов, и 5 пакетов не обновлено.
Необходимо скачать 3 974 B архивов.
После данной операции объём занятого дискового пространства возрастёт на 35,8 kB.
Пол:1 http://ru.archive.ubuntu.com/ubuntu noble/universe amd64 apt-transport-https all 2.7.14build2 [3 974 B]
Получено 3 974 B за 0с (35,9 kB/s)
Выбор ранее не выбранного пакета apt-transport-https.
(Чтение базы данных … на данный момент установлено 193307 файлов и каталогов.)
Подготовка к распаковке …/apt-transport-https_2.7.14build2_all.deb …
Распаковывается apt-transport-https (2.7.14build2) …
Настраивается пакет apt-transport-https (2.7.14build2) …
```

Импортируем PostgreSQL GPG key для верификации установочного пакета

```
daa@daa-VMware-Virtual-Platform:~$ curl -fSsL https://www.postgresql.org/media/keys/ACCC4CF8.asc | gpg --dearmor | sudo tee /usr/share/keyrings/postgresql.gpg > /dev/null
```

Импортируем APT репозиторий PostgreSQL и проапдейтим перечень репозиториев.

```
daa@daa-VMware-Virtual-Platform:~$ echo deb [arch=amd64,arm64,ppc64el signed-by=/usr/share/keyrings/postgresql.gpg] http://apt.postgresql.org/pub/repos/apt/ $(lsb_release -cs)-pgdg main | sudo tee /etc/apt/sources.list.d/postgresql.list
deb [arch=amd64,arm64,ppc64el signed-by=/usr/share/keyrings/postgresql.gpg] http://apt.postgresql.org/pub/repos/apt/ noble-pgdg main
daa@daa-VMware-Virtual-Platform:~$ sudo apt update
Сущ:1 http://ru.archive.ubuntu.com/ubuntu noble InRelease
Сущ:2 http://ru.archive.ubuntu.com/ubuntu noble-updates InRelease
Сущ:3 http://ru.archive.ubuntu.com/ubuntu noble-backports InRelease
Сущ:4 http://security.ubuntu.com/ubuntu noble-security InRelease
Пол:5 http://apt.postgresql.org/pub/repos/apt noble-pgdg InRelease [129 kB]
Сущ:6 https://download.docker.com/linux/ubuntu noble InRelease
Пол:7 http://apt.postgresql.org/pub/repos/apt noble-pgdg/main ppc64el Packages [303 kB]
Пол:8 http://apt.postgresql.org/pub/repos/apt noble-pgdg/main arm64 Packages [294 kB]
Пол:9 http://apt.postgresql.org/pub/repos/apt noble-pgdg/main amd64 Packages [306 kB]
Получено 1 032 kB за 1с (751 kB/s)
Чтение списков пакетов… Готово
Построение дерева зависимостей… Готово
Чтение информации о состоянии… Готово
Может быть обновлено 11 пакетов. Запустите «apt list --upgradable» для их показа.
```

Устанавливаем PostgreSQL 15 и убедимся что кластер запущен

```
daa@daa-VMware-Virtual-Platform:~$ sudo -u postgres pg_lsclusters
Ver Cluster Port Status Owner    Data directory              Log file
15  main    5432 online postgres /var/lib/postgresql/15/main /var/log/postgresql/postgresql-15-main.log
```

Создадим новую роль replicator с паролем ReplicatorPassword для работы с репликами. Должно совпадать с настройками Patroni из блока postgresql - authentication - replication и списком разрешённых хостов postgresql в файле pg_hba.conf:

```
daa@patroni1:~$ sudo -u postgres psql -c \
"CREATE USER replicator WITH REPLICATION ENCRYPTED PASSWORD 'QWEasd123';"
```

Установим пароль для пользователя postgres:

```
sudo -u postgres psql -c "ALTER USER postgres PASSWORD 'QWEasd123';"
```

2.2. Приступим к дальнейшей подготовке patroni

Для начала нам нужно остановить службу и удалить каталог с данными:

```
daa@patroni1:~$ sudo systemctl stop postgresql
daa@patroni1:~$ sudo rm -fr /var/lib/postgresql/15/main
```

Установим python и patroni:

```
daa@patroni1:~$ sudo apt-get install python3-pip python3-dev libpq-dev -y
daa@patroni1:~$ sudo apt-get install patroni -y
```

Создадим виртуальную среду для установки необходимых пакетов через pip

```
daa@patroni1:~$ sudo apt install python3-venv
daa@patroni1:~$ python3 -m venv .venv
```

Активируем созданную виртуальную среду

```
daa@patroni1:~$ source .venv/bin/activate
```

Установите зависимости для работы Patroni:

```
(.venv) daa@patroni1:~$ pip3 install psycopg2-binary
(.venv) daa@patroni1:~$ pip3 install wheel
(.venv) daa@patroni1:~$ pip3 install python-etcd
```

Для выхода из виртуальной среды:

```
(.venv) daa@patroni1:~$ deactivate
```

2.3. Теперь настроим конфигурацию patroni

```
daa@patroni1:~$ sudo nano /etc/patroni/config.yml
```

В конфиге указываются все необходимые параметры для развертывания кластера в т.ч. точки etcd, параметры работы кластера БД, параметры failover, настройки доступа к БД и т.д.

```
scope: patroni-cluster # одинаковое значение на всех узлах
name: patroni1 # разное значение на всех узлах
namespace: /service/ # одинаковое значение на всех узлах
log:
  level: INFO
  dir: /var/log/patroni
  file_size: 50000000
  file_num: 10

restapi:
  listen: 192.168.1.111:8008 # адрес узла, на котором находится этот файл
  connect_address: 192.168.1.111:8008 # адрес узла, на котором находится этот файл

etcd:
  hosts: 192.168.1.106:2379,192.168.1.109:2379,192.168.1.110:2379 # список всех узлов, на которых установлен etcd

bootstrap:
  method: initdb
  dcs:
    failsafe_mode: true
    ttl: 30
    loop_wait: 10
    retry_timeout: 10
    maximum_lag_on_failover: 1048576
    master_start_timeout: 300
    synchronous_mode: true
    synchronous_mode_strict: true
    synchronous_node_count: 1
    master_start_timeout: 30
    postgresql:
      use_pg_rewind: true
      use_slots: true
      parameters:
        max_connections: 2000
        superuser_reserved_connections: 5
        max_locks_per_transaction: 64
        max_prepared_transactions: 0
        huge_pages: try
        shared_buffers: 512MB
        work_mem: 128MB
        maintenance_work_mem: 256MB
        effective_cache_size: 4GB
        checkpoint_timeout: 15min
        checkpoint_completion_target: 0.9
        wal_compression: on
        min_wal_size: 2GB
        max_wal_size: 4GB
        wal_buffers: 32MB
        default_statistics_target: 1000
        seq_page_cost: 1
        random_page_cost: 4
        effective_io_concurrency: 2
        synchronous_commit: on
        autovacuum: on
        autovacuum_max_workers: 5
        autovacuum_vacuum_scale_factor: 0.01
        autovacuum_analyze_scale_factor: 0.02
        autovacuum_vacuum_cost_limit: 200
        autovacuum_vacuum_cost_delay: 20
        autovacuum_naptime: 1s
        max_files_per_process: 4096
        archive_mode: on
        archive_timeout: 1800s
        archive_command: cd .
        wal_level: replica
        wal_keep_segments: 130
        max_wal_senders: 10
        max_replication_slots: 10
        hot_standby: on
        hot_standby_feedback: True
        wal_log_hints: on
        shared_preload_libraries: pg_stat_statements,auto_explain
        pg_stat_statements.max: 10000
        pg_stat_statements.track: all
        pg_stat_statements.save: off
        auto_explain.log_min_duration: 10s
        auto_explain.log_analyze: true
        auto_explain.log_buffers: true
        auto_explain.log_timing: false
        auto_explain.log_triggers: true
        auto_explain.log_verbose: true
        auto_explain.log_nested_statements: true
        track_io_timing: on
        log_lock_waits: on
        log_temp_files: 3
        track_activities: on
        track_counts: on
        track_functions: all
        log_checkpoints: on
        logging_collector: on
        log_truncate_on_rotation: on
        log_rotation_age: 1d
        log_rotation_size: 0
        log_line_prefix: '%t [%p-%l] %r %q%u@%d '
        log_filename: 'postgresql-%a.log'
        log_directory: /var/log/postgresql

  initdb: # List options to be passed on to initdb
    - encoding: UTF8
    - locale: en_US.UTF-8
    - data-checksums

  pg_hba: # должен содержать адреса ВСЕХ машин, используемых в кластере
    - local all postgres md5
    - local all all peer
    - host all all 0.0.0.0/0 md5
    - host replication replicator localhost trust
    - host replication replicator 192.168.1.0/24 md5


postgresql:
  listen: 192.168.1.111:5432 # адрес узла, на котором находится этот файл
  connect_address: 192.168.1.111:5432 # адрес узла, на котором находится этот файл
  use_unix_socket: true
  data_dir: /var/lib/postgresql/15/main # каталог данных
  bin_dir: /usr/lib/postgresql/15/bin
  config_dir: /etc/postgresql/15/main
  pgpass: /var/lib/postgresql/.pgpass_patroni
  authentication:
    replication:
      username: postgres
      password: QWEasd123
    superuser:
      username: postgres
      password: QWEasd123
  parameters:
    unix_socket_directories: /var/run/postgresql
  pg_hba: # должен содержать адреса ВСЕХ машин, используемых в кластере
    - local all postgres md5
    - local all all peer
    - host all all 0.0.0.0/0 md5
    - host replication replicator localhost trust
    - host replication replicator 192.168.1.0/24 md5

  remove_data_directory_on_rewind_failure: false
  remove_data_directory_on_diverged_timelines: false

  create_replica_methods:
    - basebackup
  basebackup:
    max-rate: '100M'
    checkpoint: 'fast'

watchdog: 
  mode: off #Не используем, поэтому отключен
  device: /dev/watchdog
  safety_margin: 5

tags:
  nofailover: false
  noloadbalance: false
  clonefrom: false
  nosync: false
```

Сделаем пользователя postgres владельцем каталога настроек:

```
daa@patroni1:~$ sudo chown postgres:postgres -R /etc/patroni
daa@patroni1:~$ sudo chmod 700 /etc/patroni
```

2.4. Запустим кластер patroni:

```
daa@patroni1:~$ sudo systemctl start patroni
```

Проверим статус службы:

```
daa@patroni1:~$ sudo systemctl status patroni
● patroni.service - Runners to orchestrate a high-availability PostgreSQL
     Loaded: loaded (/usr/lib/systemd/system/patroni.service; enabled; preset: enabled)
     Active: active (running) since Wed 2024-12-18 14:35:09 MSK; 9s ago
   Main PID: 33819 (patroni)
      Tasks: 13 (limit: 4558)
     Memory: 164.0M (peak: 165.5M)
        CPU: 697ms
     CGroup: /system.slice/patroni.service
             ├─33819 /usr/bin/python3 /usr/bin/patroni /etc/patroni/config.yml
             ├─33835 /usr/lib/postgresql/15/bin/postgres -D /var/lib/postgresql/15/main --config-file=/etc/postgresql/15/main/postgresql.conf --listen_addresses=192.168.1.111,127.0.0.1 --port=5432 --cluster_name=patroni-cluster --wal_level=replica --hot_standby=True --max_connections=20>
             ├─33837 "postgres: patroni-cluster: logger "
             ├─33838 "postgres: patroni-cluster: checkpointer "
             ├─33839 "postgres: patroni-cluster: background writer "
             ├─33840 "postgres: patroni-cluster: startup recovering 000000020000000000000005"
             ├─33842 "postgres: patroni-cluster: walreceiver streaming 0/50001B8"
             └─33847 "postgres: patroni-cluster: postgres postgres [local] idle"

Dec 18 14:35:10 patroni1 patroni[33835]: 2024-12-18 14:35:10 MSK [33835-1]  LOG:  redirecting log output to logging collector process
Dec 18 14:35:10 patroni1 patroni[33835]: 2024-12-18 14:35:10 MSK [33835-2]  HINT:  Future log output will appear in directory "/var/log/postgresql".
Dec 18 14:35:11 patroni1 patroni[33843]: /var/run/postgresql:5432 - accepting connections
Dec 18 14:35:11 patroni1 patroni[33845]: /var/run/postgresql:5432 - accepting connections
Dec 18 14:35:11 patroni1 patroni[33819]: 2024-12-18 14:35:11,440 INFO: Lock owner: patroni2; I am patroni1
Dec 18 14:35:11 patroni1 patroni[33819]: 2024-12-18 14:35:11,440 INFO: establishing a new patroni heartbeat connection to postgres
Dec 18 14:35:11 patroni1 patroni[33819]: 2024-12-18 14:35:11,450 INFO: Local timeline=2 lsn=0/50001B8
Dec 18 14:35:11 patroni1 patroni[33819]: 2024-12-18 14:35:11,470 INFO: primary_timeline=2
Dec 18 14:35:11 patroni1 patroni[33819]: 2024-12-18 14:35:11,473 WARNING: Dropping physical replication slot patroni2 because of its xmin value 740
Dec 18 14:35:11 patroni1 patroni[33819]: 2024-12-18 14:35:11,479 INFO: no action. I am (patroni1), a secondary, and following a leader (patroni2)
```

И проверим статус кластера:

```
daa@patroni2:~$ sudo patronictl -c /etc/patroni/config.yml list
[sudo] password for daa:
+ Cluster: patroni-cluster (7449702151863418585) ---+-----------+
| Member   | Host          | Role    | State   | TL | Lag in MB |
+----------+---------------+---------+---------+----+-----------+
| patroni1 | 192.168.1.111 | Leader  | running | 43 |           |
| patroni2 | 192.168.1.112 | Replica | running | 43 |         0 |
+----------+---------------+---------+---------+----+-----------+
```

Развернем демо БД (в соответствии с описанием в документации), просто для тестов

```
daa@patroni1:~$ sudo psql -f demo_big.sql -U postgres
```

#### 3. Установим pgbouncer на обе ноды

PGBouncer — программа, управляющая пулом соединений PostgreSQL. Любое конечное приложение может подключиться к PGBouncer, как если бы это был непосредственно сервер PostgreSQL. PGBouncer создаст подключение к реальному серверу, либо задействует одно из ранее установленных подключений.

Назначение PGBouncer — минимизировать издержки, связанные с установлением новых подключений к PostgreSQL.

```
daa@patroni1:~$ sudo apt-get install pgbouncer -y
```

Переместите конфигурационный файл по умолчанию:

```
daa@patroni1:/etc$ sudo mv /etc/pgbouncer/pgbouncer.ini{,.original}
```

Создадим и отредактируем конфигурационный файл. :

```
sudo nano /etc/pgbouncer/pgbouncer.ini
```

```
  GNU nano 7.2                                      /etc/pgbouncer/pgbouncer.ini
[databases]
* = host=192.168.1.111,192.168.1.112 port=5432

[pgbouncer]
logfile = /var/log/postgresql/pgbouncer.log
pidfile = /var/run/postgresql/pgbouncer.pid
listen_addr = *
listen_port = 6432
unix_socket_dir = /var/run/postgresql
auth_type = md5
auth_file = /etc/pgbouncer/userlist.txt
admin_users = postgres
ignore_startup_parameters = extra_float_digits,geqo
pool_mode = session
server_reset_query = DISCARD ALL
max_client_conn = 10000
default_pool_size = 800
reserve_pool_size = 150
reserve_pool_timeout = 1
max_db_connections = 1000
pkt_buf = 8192
listen_backlog = 4096
log_connections = 0
log_disconnections = 0
```

Параметры конфига:

   - listen_addr — список адресов через запятую, где прослушивать соединения TCP. Если вы установите *, будут прослушивать все адреса;
   - listen_port — порт для прослушивания, по умолчанию установлено 6432;
   - pool_mode — режим работы;
   - auth_type — режим аутентификации пользователей;
   - max_client_conn — максимально допустимое количество клиентских подключений;
   - default_pool_size — размер пула открытых подключений к базам данных;
   - reserve_pool_size — размер резервного пула открытых подключений к базам данных;
   - max_db_connections — максимально допустимое количество открытых подключений к базам данных;
   - [databases] — определяет имена баз данных, к которым могут подключаться клиенты PgBouncer. Указывает, куда будут маршрутизироваться эти подключения.

Добавим в файл /etc/pgbouncer/userlist.txt имена пользователей и пароли, с которыми PGBouncer подключается к базе: 

```
daa@patroni1:/etc$  sudo nano /etc/pgbouncer/userlist.txt
```

В нашем случае это пользователь postgres

```
 GNU nano 7.2                                       /etc/pgbouncer/userlist.txt
"postgres" "QWEasd123"
```

Перезапустите PGBouncer, выполнив команду:

```
daa@patroni1:/etc$  sudo systemctl daemon-reload
daa@patroni1:/etc$  sudo systemctl restart pgbouncer
```

Проверим статус 

```
daa@patroni1:/etc$ sudo systemctl status pgbouncer
● pgbouncer.service - connection pooler for PostgreSQL
     Loaded: loaded (/usr/lib/systemd/system/pgbouncer.service; enabled; preset: enabled)
     Active: active (running) since Thu 2024-12-19 05:20:10 MSK; 7s ago
       Docs: man:pgbouncer(1)
             https://www.pgbouncer.org/
   Main PID: 137160 (pgbouncer)
      Tasks: 2 (limit: 4558)
     Memory: 1.2M (peak: 1.4M)
        CPU: 12ms
     CGroup: /system.slice/pgbouncer.service
             └─137160 /usr/sbin/pgbouncer /etc/pgbouncer/pgbouncer.ini

Dec 19 05:20:10 patroni1 systemd[1]: Starting pgbouncer.service - connection pooler for PostgreSQL...
Dec 19 05:20:10 patroni1 pgbouncer[137160]: kernel file descriptor limit: 1500 (hard: 1500); max_client_conn: 10000, max expect>
Dec 19 05:20:10 patroni1 pgbouncer[137160]: listening on 0.0.0.0:6432
Dec 19 05:20:10 patroni1 pgbouncer[137160]: listening on [::]:6432
Dec 19 05:20:10 patroni1 pgbouncer[137160]: listening on unix:/var/run/postgresql/.s.PGSQL.6432
Dec 19 05:20:10 patroni1 pgbouncer[137160]: process up: PgBouncer 1.23.1, libevent 2.1.12-stable (epoll), adns: c-ares 1.27.0, >
Dec 19 05:20:10 patroni1 systemd[1]: Started pgbouncer.service - connection pooler for PostgreSQL.
```

Проверим подключение к БД через pgbouncer

```
daa@patroni1:/etc$ c\demo
Password for user postgres:
psql (15.10 (Ubuntu 15.10-1.pgdg24.04+1))
Type "help" for help.

postgres=# \c demo
You are now connected to database "demo" as user "postgres".
demo=# \q
```

#### 4. Развернем haproxy+keepalived

- haproxy1 192.168.1.113
- haproxy1 192.168.1.114
- VIP 192.168.1.115

Keepalived — программный инструмент, предназначенный для обеспечения высокой доступности и отказоустойчивости сетевых сервисов. 
VIP (Virtual IP Address) — это виртуальный IP-адрес, который не привязан к какому-то конкретному физическому интерфейсу. VIP может быть использован для того, чтобы обеспечить доступ к сервису, который работает на нескольких серверах (например, в кластере, как в нашем случае).
HAProxy — инструмент обеспечения высокой доступности и балансировки нагрузки. С его помощью автоматически проверяется порт 8008 сервиса Patroni на серверах PostgreSQL с ролью master.

Трафик операций в кластер распределяется следующим образом:

операции записи, приходящие на haproxy-cluster:5000, направляются на сервер с ролью master;
операции чтения, приходящие на haproxy-cluster:5001, направляются на сервера с ролью slave.
В случае сбоя весь трафик будет перенаправлен в кластер Master-Replica(s), т. е. операции записи и чтения начнут поступать именно сюда.

4.1. Настроим VIP для кластера haproxy

Изменим конфигурацию интерфейса на обеих нодах и добавим ip адрес 192.168.1.115 к интерфейсам.

```
daa@haproxy1:/etc/netplan$ sudo nano 90-NM-14f59568-5076-387a-aef6-10adfcca2e26.yaml
```

Для этого отредактируем конфигурационных файл в ОС 

```
  GNU nano 7.2                             90-NM-14f59568-5076-387a-aef6-10adfcca2e26.yaml
network:
  version: 2
  ethernets:
    ens33:
      renderer: NetworkManager
      match:
        macaddress: "00:0C:29:6D:91:97"
      addresses:
      - "192.168.1.113/24"
      - "192.168.1.115/24"
      nameservers:
        addresses:
        - 8.8.8.8
      networkmanager:
        uuid: "14f59568-5076-387a-aef6-10adfcca2e26"
        name: "netplan-ens33"
        passthrough:
          connection.timestamp: "1734969280"
          ipv4.address1: "192.168.1.113/24,192.168.1.1"
          ipv4.method: "manual"
          ipv6.method: "disabled"
          ipv6.ip6-privacy: "-1"
          proxy._: ""
```

А так же занесем в файл hosts данные по ip адресу кластера haproxy-cluster:

```
daa@haproxy1:/etc/netplan$ sudo nano /etc/hosts
```

```

 GNU nano 7.2                                               /etc/hosts
127.0.0.1 localhost
127.0.1.1 daa-pgsql

# The following lines are desirable for IPv6 capable hosts
::1     ip6-localhost ip6-loopback
fe00::0 ip6-localnet
ff00::0 ip6-mcastprefix
ff02::1 ip6-allnodes
ff02::2 ip6-allrouters

192.168.1.106   etcd1
192.168.1.109   etcd2
192.168.1.110   etcd3
192.168.1.111   patroni1
192.168.1.112   patroni2
192.168.1.113   haproxy1
192.168.1.114   haproxy2
192.168.1.115   haproxy-cluster
```

Включим параметры ядра net.ipv4.ip_nonlocal_bind, net.ipv4.ip_forward, выполнив следующие команды:

```
daa@haproxy1:/etc/netplan$ sudo echo net.ipv4.ip_nonlocal_bind=1 >> /etc/sysctl.conf
daa@haproxy1:/etc/netplan$ sudo echo net.ipv4.ip_forward=1 >> /etc/sysctl.conf
daa@haproxy1:/etc/netplan$ sudo sysctl -p
```

4.2. Установим Haproxy и keepalived

Haproxy:

```
daa@haproxy1:/etc/netplan$ sudo apt install haproxy -y
```

keepalived:

```
daa@haproxy1:/etc/netplan$ sudo apt update
daa@haproxy1:/etc/netplan$ sudo apt install keepalived
```

4.3. Настройка keepalived
   
Создадим директорию keepalived:

```
daa@haproxy1:/etc/netplan$ sudo mkdir -p /usr/libexec/keepalived
```

Создадим файл скрипта haproxy_check.sh для отслеживания работоспособности HAProxy:

```
daa@haproxy1:/etc/netplan$ sudo nano /usr/libexec/keepalived/haproxy_check.sh
```

Добавим в файл haproxy_check.sh скрипт проверки работоспособности HAProxy:

```
#!/bin/bash
/bin/kill -0 $(cat /var/run/haproxy.pid)
```

Создадим файл конфигурации keepalived на каждой ноде:

```
daa@haproxy1:/etc/netplan$ sudo nano /etc/keepalived/keepalived.conf
```

Добавим в файл keepalived.conf пример конфигураций keepalived для каждого узла, там же укажем VIP:

```
  GNU nano 7.2                                     /etc/keepalived/keepalived.conf
global_defs {
  router_id haproxy1
  enable_script_security
  script_user root
}

vrrp_script haproxy_check {
  script "/usr/libexec/keepalived/haproxy_check.sh"
  interval 2
  weight 2
}

vrrp_instance VI_1 {
  interface ens33
  virtual_router_id 101
  priority 100
  advert_int 2
  state MASTER
  virtual_ipaddress {
    192.168.1.115
  }
  track_script {
    haproxy_check
  }
  authentication {
    auth_type PASS
    auth_pass SecretPassword
  }
}
```

Где введённые параметры:

  router_id — ID роутера, уникальное значение на каждом узле;
  script_user — пользователь, от имени которого будут запускаться скрипты VRRP;
  interface — наименование интерфейса, к которому будет привязан keepalived;
  virtual_router_id — ID виртуального роутера, общее значение на всех узлах;
  priority — приоритет нод внутри виртуального роутера;
  state — тип роли ноды в виртуальном роутере;
  virtual_ipaddress — виртуальный IP;
  auth_type — тип аутентификации в виртуальном роутере;
  auth_pass — пароль.


4.4. Настроим haproxy

Переместим конфигурационный файл по умолчанию:

```
daa@haproxy1:/etc/netplan$ sudo mv /etc/haproxy/haproxy.cfg{,.original}
```

Откроем для редактирования конфигурационный файл:

```
daa@haproxy1:/etc/netplan$ sudo nano /etc/haproxy/haproxy.cfg
```

И добавим в него конфигурацию:

```
  GNU nano 7.2                                        /etc/haproxy/haproxy.cfg
global
    maxconn 100000
    log 127.0.0.1:514 local0
    log 127.0.0.1:514 local1 notice
    chroot /var/lib/haproxy
    stats socket /run/haproxy/admin.sock mode 660 level admin expose-fd listeners
    stats timeout 30s
    user postgres
    group postgres
    daemon

defaults
    mode tcp
    log global
    retries 2
    timeout queue 5s
    timeout connect 5s
    timeout client 60m
    timeout server 60m
    timeout check 15s

listen stats
    mode http
    bind haproxy-cluster:7000
    stats enable
    stats uri /

### PostgreSQL ###
listen postgres_master
    bind *:5000
    option tcplog
    option httpchk OPTIONS /master
    http-check expect status 200
    default-server inter 3s fastinter 1s fall 3 rise 4 on-marked-down shutdown-sessions
    server patroni1 patroni1:6432 check port 8008
    server patroni2 patroni2:6432 check port 8008

listen postgres_replicas
    bind *:5001
    option tcplog
    option httpchk OPTIONS /health
    balance roundrobin
    http-check expect status 200
    default-server inter 3s fastinter 1s fall 3 rise 2 on-marked-down shutdown-sessions
    server patroni1 patroni1:6432 check port 8008
    server patroni2 patroni2:6432 check port 8008
### PostgreSQL ###
```

Перезапустим keepalived и Haproxy на всех узлах:

```
daa@haproxy1:/etc/netplan$ sudo systemctl restart keepalived
daa@haproxy1:/etc/netplan$ sudo systemctl restart haproxy
```

4.5. Проверим результат.

Проверим статус keepalived:

```
daa@haproxy1:/etc/netplan$ sudo systemctl status keepalived
● keepalived.service - Keepalive Daemon (LVS and VRRP)
     Loaded: loaded (/usr/lib/systemd/system/keepalived.service; enabled; preset: enabled)
     Active: active (running) since Wed 2024-12-25 18:55:43 MSK; 12s ago
       Docs: man:keepalived(8)
             man:keepalived.conf(5)
             man:genhash(1)
             https://keepalived.org
   Main PID: 2961 (keepalived)
      Tasks: 2 (limit: 4558)
     Memory: 1.8M (peak: 2.1M)
        CPU: 8ms
     CGroup: /system.slice/keepalived.service
             ├─2961 /usr/sbin/keepalived --dont-fork
             └─2964 /usr/sbin/keepalived --dont-fork

Dec 25 18:55:43 haproxy1 Keepalived[2961]: Starting VRRP child process, pid=2964
Dec 25 18:55:43 haproxy1 Keepalived_vrrp[2964]: (/etc/keepalived/keepalived.conf: Line 27) Truncating auth_pass to 8 characters
Dec 25 18:55:43 haproxy1 Keepalived_vrrp[2964]: WARNING - script '/usr/libexec/keepalived/haproxy_check.sh' is not executable f>
Dec 25 18:55:43 haproxy1 Keepalived[2961]: Startup complete
Dec 25 18:55:43 haproxy1 systemd[1]: Started keepalived.service - Keepalive Daemon (LVS and VRRP).
Dec 25 18:55:43 haproxy1 Keepalived_vrrp[2964]: (VI_1) Entering BACKUP STATE (init)
Dec 25 18:55:44 haproxy1 Keepalived_vrrp[2964]: (VI_1) received lower priority (90) advert from 192.168.1.114 - discarding
Dec 25 18:55:46 haproxy1 Keepalived_vrrp[2964]: (VI_1) received lower priority (90) advert from 192.168.1.114 - discarding
Dec 25 18:55:48 haproxy1 Keepalived_vrrp[2964]: (VI_1) received lower priority (90) advert from 192.168.1.114 - discarding
Dec 25 18:55:49 haproxy1 Keepalived_vrrp[2964]: (VI_1) Entering MASTER STATE
```

Проверим статус Haproxy:

```
daa@haproxy1:/etc/netplan$ sudo systemctl status haproxy
● haproxy.service - HAProxy Load Balancer
     Loaded: loaded (/usr/lib/systemd/system/haproxy.service; enabled; preset: enabled)
     Active: active (running) since Wed 2024-12-25 16:23:28 MSK; 5h 59min ago
       Docs: man:haproxy(1)
             file:/usr/share/doc/haproxy/configuration.txt.gz
   Main PID: 1321 (haproxy)
     Status: "Ready."
      Tasks: 3 (limit: 4558)
     Memory: 23.0M (peak: 23.5M)
        CPU: 8.085s
     CGroup: /system.slice/haproxy.service
             ├─1321 /usr/sbin/haproxy -Ws -f /etc/haproxy/haproxy.cfg -p /run/haproxy.pid -S /run/haproxy-master.sock
             └─1373 /usr/sbin/haproxy -Ws -f /etc/haproxy/haproxy.cfg -p /run/haproxy.pid -S /run/haproxy-master.sock

Dec 25 18:13:11 haproxy1 haproxy[1373]: [ALERT]    (1373) : proxy 'postgres_master' has no server available!
Dec 25 18:13:11 haproxy1 haproxy[1373]: [WARNING]  (1373) : Server postgres_replicas/patroni2 is DOWN, reason: Layer7 wrong sta>
Dec 25 18:13:14 haproxy1 haproxy[1373]: [WARNING]  (1373) : Server postgres_master/patroni1 is UP, reason: Layer7 check passed,>
Dec 25 18:13:15 haproxy1 haproxy[1373]: [WARNING]  (1373) : Server postgres_replicas/patroni2 is UP, reason: Layer7 check passe>
Dec 25 18:15:14 haproxy1 haproxy[1373]: [WARNING]  (1373) : Server postgres_master/patroni1 is DOWN, reason: Layer7 wrong statu>
Dec 25 18:15:14 haproxy1 haproxy[1373]: [ALERT]    (1373) : proxy 'postgres_master' has no server available!
Dec 25 18:15:17 haproxy1 haproxy[1373]: [WARNING]  (1373) : Server postgres_master/patroni2 is UP, reason: Layer7 check passed,>
Dec 25 18:18:41 haproxy1 haproxy[1373]: [WARNING]  (1373) : Server postgres_master/patroni2 is DOWN, reason: Layer7 wrong statu>
Dec 25 18:18:41 haproxy1 haproxy[1373]: [ALERT]    (1373) : proxy 'postgres_master' has no server available!
Dec 25 18:18:42 haproxy1 haproxy[1373]: [WARNING]  (1373) : Server postgres_master/patroni1 is UP, reason: Layer7 check passed,>
lines 1-24/24 (END)
```

Проверим подключение к БД через haproxy

Подключение к мастеру

```
daa@haproxy1:~$ psql -h 192.168.1.115 -U postgres -d demo -p 5000
Password for user postgres:
psql (15.9 (Ubuntu 15.9-1.pgdg24.04+1), server 15.10 (Ubuntu 15.10-1.pgdg24.04+1))
Type "help" for help.

demo=#
```

Подключение к реплике

```
daa@haproxy2:/dev$ psql -h 192.168.1.115 -U postgres -d demo -p 5001
Password for user postgres:
psql (15.9 (Ubuntu 15.9-1.pgdg24.04+1), server 15.10 (Ubuntu 15.10-1.pgdg24.04+1))
Type "help" for help.

demo=#
```

Посмотрим результат в веб интерфейсе Haproxy

http://192.168.1.115:7000/

![image](https://github.com/user-attachments/assets/21fd9eaa-aecb-4275-b1d2-c226b7861c1e)

Все компоненты работают. Можно переходить к тестированию.

#### 5. Протестируем то, что получилось

5.1. Подготовка

Попробуем протестировать, то что у нас получилось, для тестирования будем использовать hatester

```
https://github.com/jobinau/pgscripts/blob/main/patroni/HAtester.py
```

Установим драйвер и скачаем скрипт

```
sudo apt-get install python3-psycopg2
curl -LO https://raw.githubusercontent.com/jobinau/pgscripts/main/patroni/HAtester.py
chmod +x HAtester.py
```

Отредактируем его конфигурацию

```
# CONNECTION DETAILS
host = "192.168.1.115"
dbname = "demo"
user = "postgres"
password = "QWEasd123"
```

Создадим в БД необходимую для тестов таблицу

```
CREATE TABLE HATEST (TM TIMESTAMP);
```

Запустим тестирование:

На запись:

```
./HAtester.py 5000
```

```
Working with:   MASTER - 192.168.1.112
     Inserted: 2024-12-23 13:49:09.030794

 Working with:   MASTER - 192.168.1.112
     Inserted: 2024-12-23 13:49:10.033195

 Working with:   MASTER - 192.168.1.112
     Inserted: 2024-12-23 13:49:11.036122

 Working with:   MASTER - 192.168.1.112
     Inserted: 2024-12-23 13:49:12.039146
```

На чтение:

```
./HAtester.py 5001
```

```
 Working with:    REPLICA - 192.168.1.111
     Retrived: 2024-12-23 13:49:28.088576

 Working with:    REPLICA - 192.168.1.111
     Retrived: 2024-12-23 13:49:29.091379

 Working with:    REPLICA - 192.168.1.111
     Retrived: 2024-12-23 13:49:30.094587

 Working with:    REPLICA - 192.168.1.111
     Retrived: 2024-12-23 13:49:31.097560
```

5.1. Потеря сетевой связи с Replica

Отключим сетевой интерфейс ВМ реплики, на данный момент это patroni1

```
+ Cluster: patroni-cluster (7449702151863418585) +----+-----------+
| Member   | Host          | Role    | State     | TL | Lag in MB |
+----------+---------------+---------+-----------+----+-----------+
| patroni1 | 192.168.1.111 | Replica | streaming | 20 |         0 |
| patroni2 | 192.168.1.112 | Leader  | running   | 20 |           |
+----------+---------------+---------+-----------+----+-----------+
```

Для операций чтения произошло автоматическое переключение на мастер ноду:

```
 Working with:    REPLICA - 192.168.1.111
     Retrived: 2024-12-24 14:56:20.123431

Trying to connect
 Working with:   MASTER - 192.168.1.112
     Inserted: 2024-12-24 14:56:35.042064

 Working with:   MASTER - 192.168.1.112
     Inserted: 2024-12-24 14:56:36.046823

```

При этом в логах haproxy:

```
2024-12-24T14:56:33.970563+03:00 haproxy1 haproxy[2455]: [WARNING]  (2455) : Server postgres_replicas/patroni1 is DOWN, reason: Layer4 timeout, check duration: 3002ms. 1 active and 0 backup servers left. 1 sessions active, 0 requeued, 0 remaining in queue.
```

В patroni на второй ноде:

```
daa@patroni2:~$ sudo patronictl -c /etc/patroni/config.yml list
+ Cluster: patroni-cluster (7449702151863418585) --+-----------+
| Member   | Host          | Role   | State   | TL | Lag in MB |
+----------+---------------+--------+---------+----+-----------+
| patroni2 | 192.168.1.112 | Leader | running | 20 |           |
+----------+---------------+--------+---------+----+-----------+
```

Включим интерфейс обратно:

haproxy:

```
2024-12-24T15:00:23.247611+03:00 haproxy1 haproxy[2455]: [WARNING]  (2455) : Server postgres_replicas/patroni1 is UP, reason: Layer7 check passed, code: 200, check duration: 1ms. 2 active and 0 backup servers online. 0 sessions requeued, 0 total in queue.
```

patroni2:

```
daa@patroni2:~$ sudo patronictl -c /etc/patroni/config.yml list
+ Cluster: patroni-cluster (7449702151863418585) +----+-----------+
| Member   | Host          | Role    | State     | TL | Lag in MB |
+----------+---------------+---------+-----------+----+-----------+
| patroni1 | 192.168.1.111 | Replica | streaming | 20 |         0 |
| patroni2 | 192.168.1.112 | Leader  | running   | 20 |           |
+----------+---------------+---------+-----------+----+-----------+
```

patroni1:

```
2024-12-24 15:00:21,403 ERROR: Failed to get list of machines from http://192.168.1.106:2379/v2: MaxRetryError("HTTPConnectionPo
ol(host='192.168.1.106', port=2379): Max retries exceeded with url: /v2/machines (Caused by NewConnectionError('<urllib3.connect
ion.HTTPConnection object at 0x73b31412b860>: Failed to establish a new connection: [Errno 101] Network is unreachable'))")
2024-12-24 15:00:21,403 ERROR: Could not get the list of servers, maybe you provided the wrong host(s) to connect to?
2024-12-24 15:00:22,426 INFO: no action. I am (patroni1), a secondary, and following a leader (patroni2)
2024-12-24 15:00:26,393 INFO: no action. I am (patroni1), a secondary, and following a leader (patroni2)
```

5.2. Потеря сетевой связанности с Master

Отключим сетевой интерфейс ВМ мастера, на данный момент это patroni2

```
+ Cluster: patroni-cluster (7449702151863418585) +----+-----------+
| Member   | Host          | Role    | State     | TL | Lag in MB |
+----------+---------------+---------+-----------+----+-----------+
| patroni1 | 192.168.1.111 | Replica | streaming | 20 |         0 |
| patroni2 | 192.168.1.112 | Leader  | running   | 20 |           |
+----------+---------------+---------+-----------+----+-----------+
```

Для операций чтения, произошла задержка в операциях, пока нода реплика не взяла на себя роль мастера.

```
 Working with:    REPLICA - 192.168.1.111
     Retrived: 2024-12-24 17:51:02.142796

 Working with:   MASTER - 192.168.1.111
Trying to connect
 Working with:   MASTER - 192.168.1.111
     Inserted: 2024-12-24 17:51:49.967949
```

Операции записи остановились, до момена запуска мастер ноды на реплике

```
 Working with:   MASTER - 192.168.1.112
     Inserted: 2024-12-24 17:51:02.142796

Trying to connect
Unable to connect to database :
connection to server at "192.168.1.115", port 5000 failed: server closed the connection unexpectedly
        This probably means the server terminated abnormally
        before or while processing the request.

daa@test1:~$ ./HAtester.py 5000
 Working with:   MASTER - 192.168.1.111
     Inserted: 2024-12-24 17:52:04.977101

 Working with:   MASTER - 192.168.1.111
     Inserted: 2024-12-24 17:52:05.980919
```


haproxу, видим сообщения о недоступности patroni2 и то как patroni1 становится мастером:

```
2024-12-24T17:28:54.113355+03:00 haproxy1 haproxy[1350]: [WARNING]  (1350) : Server postgres_master/patroni2 is UP, reason: Layer7 check passed, code: 200, check duration: 1ms. 1 active and 0 backup servers online. 0 sessions requeued, 0 total in queue.
2024-12-24T17:28:57.100660+03:00 haproxy1 haproxy[1350]: [WARNING]  (1350) : Server postgres_replicas/patroni1 is UP, reason: Layer7 check passed, code: 200, check duration: 1ms. 2 active and 0 backup servers online. 0 sessions requeued, 0 total in queue.
2024-12-24T17:29:33.146709+03:00 haproxy1 haproxy[1350]: [WARNING]  (1350) : Server postgres_replicas/patroni2 is DOWN, reason: Layer4 timeout, check duration: 3003ms. 1 active and 0 backup servers left. 0 sessions active, 0 requeued, 0 remaining in queue.
2024-12-24T17:29:35.176886+03:00 haproxy1 haproxy[1350]: [WARNING]  (1350) : Server postgres_master/patroni2 is DOWN, reason: Layer4 timeout, check duration: 3004ms. 0 active and 0 backup servers left. 0 sessions active, 0 requeued, 0 remaining in queue.
2024-12-24T17:29:35.176999+03:00 haproxy1 haproxy[1350]: [ALERT]    (1350) : proxy 'postgres_master' has no server available!
2024-12-24T17:29:59.717682+03:00 haproxy1 haproxy[1350]: [WARNING]  (1350) : Server postgres_master/patroni1 is UP, reason: Layer7 check passed, code: 200, check duration: 1ms. 1 active and 0 backup servers online. 0 sessions requeued, 0 total in queue.
```

![image](https://github.com/user-attachments/assets/a1f9793b-06b4-41c8-aba4-b1ddc4210eeb)

patroni1, переключение в лидера:

```
daa@patroni1:~$ sudo patronictl -c /etc/patroni/config.yml list
+ Cluster: patroni-cluster (7449702151863418585) --+-----------+
| Member   | Host          | Role   | State   | TL | Lag in MB |
+----------+---------------+--------+---------+----+-----------+
| patroni1 | 192.168.1.111 | Leader | running | 32 |           |
+----------+---------------+--------+---------+----+-----------+
```

```
2024-12-26 23:30:43,300 INFO: no action. I am (patroni2), a secondary, and following a leader (patroni1)
2024-12-26 23:30:53,342 INFO: no action. I am (patroni2), a secondary, and following a leader (patroni1)
2024-12-26 23:31:05,030 WARNING: Request failed to patroni1: GET http://192.168.1.111:8008/patroni (HTTPConnectionPool(host='192.168.1.111', port=8008): Max retries exceeded with url: /patroni (Caused by ConnectTimeoutError(<urllib3.connection.HTTPConnection object at 0x778710584650>, 'Connection to 192.168.1.111 timed out. (connect timeout=2)')))
2024-12-26 23:31:05,162 INFO: promoted self to leader by acquiring session lock
2024-12-26 23:31:05,162 INFO: Lock owner: patroni2; I am patroni2
2024-12-26 23:31:05,250 INFO: updated leader lock during promote
2024-12-26 23:31:06,301 INFO: no action. I am (patroni2), the leader with the lock
2024-12-26 23:31:16,170 INFO: Lock owner: patroni2; I am patroni2
2024-12-26 23:31:16,259 INFO: Dropped unknown replication slot 'patroni1'
2024-12-26 23:31:16,303 INFO: no action. I am (patroni2), the leader with the lock
2024-12-26 23:31:26,256 INFO: no action. I am (patroni2), the leader with the lock

```

Теперь включим ноду patroni2 обратно:

```
daa@patroni1:~$ sudo patronictl -c /etc/patroni/config.yml list
+ Cluster: patroni-cluster (7449702151863418585) +----+-----------+
| Member   | Host          | Role    | State     | TL | Lag in MB |
+----------+---------------+---------+-----------+----+-----------+
| patroni1 | 192.168.1.111 | Leader  | running   | 32 |           |
| patroni2 | 192.168.1.112 | Replica | start failed  | 32 |         0 |
+----------+---------------+---------+-----------+----+-----------+
```

```
daa@patroni1:~$ sudo patronictl -c /etc/patroni/config.yml list
+ Cluster: patroni-cluster (7449702151863418585) +----+-----------+
| Member   | Host          | Role    | State     | TL | Lag in MB |
+----------+---------------+---------+-----------+----+-----------+
| patroni1 | 192.168.1.111 | Leader  | running   | 32 |           |
| patroni2 | 192.168.1.112 | Replica | streaming | 32 |         0 |
+----------+---------------+---------+-----------+----+-----------+
```

haproxу, видим, что нода восстановилась уже как репилка:

```
2024-12-24T17:42:56.268815+03:00 haproxy1 haproxy[1350]: [WARNING]  (1350) : Server postgres_replicas/patroni2 is UP, reason: Layer7 check passed, code: 200, check duration: 1ms. 2 active and 0 backup servers online. 0 sessions requeued, 0 total in queue.
```

![image](https://github.com/user-attachments/assets/bdde159f-22b9-48ad-a2ae-606f157f3df0)

По логам patroni2 видим весь процесс восстановления, в т.ч. запуск pg_rewind и выбор etcd сервера

```
2024-12-26 23:42:09,798 ERROR: Failed to get list of machines from http://192.168.1.109:2379/v3: MaxRetryError("HTTPConnectionPool(host='192.168.1.109', port=2379): Max retries exceeded with url: /v3/cluster/member/list (Caused by NewConnectionError('<urllib3.connection.HTTPConnection object at 0x7a542e39f0e0>: Failed to establish a new connection: [Errno 101] Network is unreachable'))")
2024-12-26 23:42:09,798 ERROR: KVCache.run EtcdException('Could not get the list of servers, maybe you provided the wrong host(s) to connect to?')
2024-12-26 23:42:10,931 WARNING: Postgresql is not running.
2024-12-26 23:42:10,931 INFO: Lock owner: patroni1; I am patroni2
2024-12-26 23:42:10,933 INFO: pg_controldata:
  pg_control version number: 1300
  Catalog version number: 202209061
  Database system identifier: 7449702151863418585
  Database cluster state: shut down
  pg_control last modified: Thu Dec 26 23:40:43 2024
  Latest checkpoint location: 0/AD000028
  Latest checkpoint's REDO location: 0/AD000028
  Latest checkpoint's REDO WAL file: 0000003200000000000000AD
  Latest checkpoint's TimeLineID: 50
  Latest checkpoint's PrevTimeLineID: 50
  Latest checkpoint's full_page_writes: on
  Latest checkpoint's NextXID: 0:29946
  Latest checkpoint's NextOID: 49243
  Latest checkpoint's NextMultiXactId: 1
  Latest checkpoint's NextMultiOffset: 0
  Latest checkpoint's oldestXID: 717
  Latest checkpoint's oldestXID's DB: 1
  Latest checkpoint's oldestActiveXID: 0
  Latest checkpoint's oldestMultiXid: 1
  Latest checkpoint's oldestMulti's DB: 1
  Latest checkpoint's oldestCommitTsXid: 0
  Latest checkpoint's newestCommitTsXid: 0
  Time of latest checkpoint: Thu Dec 26 23:40:43 2024
  Fake LSN counter for unlogged rels: 0/3E8
  Minimum recovery ending location: 0/0
  Min recovery ending loc's timeline: 0
  Backup start location: 0/0
  Backup end location: 0/0
  End-of-backup record required: no
  wal_level setting: replica
  wal_log_hints setting: on
  max_connections setting: 2000
  max_worker_processes setting: 8
  max_wal_senders setting: 10
  max_prepared_xacts setting: 0
  max_locks_per_xact setting: 64
  track_commit_timestamp setting: off
  Maximum data alignment: 8
  Database block size: 8192
  Blocks per segment of large relation: 131072
  WAL block size: 8192
  Bytes per WAL segment: 16777216
  Maximum length of identifiers: 64
  Maximum columns in an index: 32
  Maximum size of a TOAST chunk: 1996
  Size of a large-object chunk: 2048
  Date/time type storage: 64-bit integers
  Float8 argument passing: by value
  Data page checksum version: 1
  Mock authentication nonce: f6dfe4464252d43f17386483dfdaf87961a5633d17ee3386e6e60a1504df09e8

2024-12-26 23:42:10,933 INFO: Lock owner: patroni1; I am patroni2
2024-12-26 23:42:10,939 INFO: Local timeline=50 lsn=0/AD000028
2024-12-26 23:42:10,959 INFO: primary_timeline=51
2024-12-26 23:42:10,959 INFO: primary: history=47       0/AA0001F0      no recovery target specified
48      0/AB0000A0      no recovery target specified
49      0/AC0000A0      no recovery target specified
50      0/AC0001B8      no recovery target specified
2024-12-26 23:42:10,965 INFO: running pg_rewind from patroni1
2024-12-26 23:42:10,993 INFO: running pg_rewind from dbname=postgres user=postgres host=192.168.1.111 port=5432 target_session_attrs=read-write
2024-12-26 23:42:17,141 INFO: Lock owner: patroni1; I am patroni2
2024-12-26 23:42:17,141 INFO: running pg_rewind from patroni1 in progress
2024-12-26 23:42:27,141 INFO: Lock owner: patroni1; I am patroni2
2024-12-26 23:42:27,186 INFO: running pg_rewind from patroni1 in progress
2024-12-26 23:42:29,720 INFO: Lock owner: patroni1; I am patroni2
2024-12-26 23:42:29,720 INFO: running pg_rewind from patroni1 in progress
2024-12-26 23:42:39,720 INFO: Lock owner: patroni1; I am patroni2
2024-12-26 23:42:39,767 INFO: running pg_rewind from patroni1 in progress
2024-12-26 23:42:49,721 INFO: Lock owner: patroni1; I am patroni2
2024-12-26 23:42:49,721 INFO: running pg_rewind from patroni1 in progress
2024-12-26 23:42:59,721 INFO: Lock owner: patroni1; I am patroni2
2024-12-26 23:42:59,763 INFO: running pg_rewind from patroni1 in progress
2024-12-26 23:43:01,741 INFO: pg_rewind exit code=0
2024-12-26 23:43:01,741 INFO:  stdout=
2024-12-26 23:43:01,741 INFO:  stderr=pg_rewind: servers diverged at WAL location 0/AC0001B8 on timeline 50
pg_rewind: rewinding from last common checkpoint at 0/AC000108 on timeline 50
pg_rewind: Done!

2024-12-26 23:43:01,742 WARNING: Postgresql is not running.
2024-12-26 23:43:01,742 INFO: Lock owner: patroni1; I am patroni2
2024-12-26 23:43:02,197 INFO: pg_controldata:
  pg_control version number: 1300
  Catalog version number: 202209061
  Database system identifier: 7449702151863418585
  Database cluster state: in archive recovery
  pg_control last modified: Thu Dec 26 23:43:00 2024
  Latest checkpoint location: 0/AC000220
  Latest checkpoint's REDO location: 0/AC0001E8
  Latest checkpoint's REDO WAL file: 0000003300000000000000AC
  Latest checkpoint's TimeLineID: 51
  Latest checkpoint's PrevTimeLineID: 51
  Latest checkpoint's full_page_writes: on
  Latest checkpoint's NextXID: 0:29946
  Latest checkpoint's NextOID: 49243
  Latest checkpoint's NextMultiXactId: 1
  Latest checkpoint's NextMultiOffset: 0
  Latest checkpoint's oldestXID: 717
  Latest checkpoint's oldestXID's DB: 1
  Latest checkpoint's oldestActiveXID: 29946
  Latest checkpoint's oldestMultiXid: 1
  Latest checkpoint's oldestMulti's DB: 1
  Latest checkpoint's oldestCommitTsXid: 0
  Latest checkpoint's newestCommitTsXid: 0
  Time of latest checkpoint: Thu Dec 26 23:40:58 2024
  Fake LSN counter for unlogged rels: 0/3E8
  Minimum recovery ending location: 0/AC0002D0
  Min recovery ending loc's timeline: 51
  Backup start location: 0/0
  Backup end location: 0/0
  End-of-backup record required: no
  wal_level setting: replica
  wal_log_hints setting: on
  max_connections setting: 2000
  max_worker_processes setting: 8
  max_wal_senders setting: 10
  max_prepared_xacts setting: 0
  max_locks_per_xact setting: 64
  track_commit_timestamp setting: off
  Maximum data alignment: 8
  Database block size: 8192
  Blocks per segment of large relation: 131072
  WAL block size: 8192
  Bytes per WAL segment: 16777216
  Maximum length of identifiers: 64
  Maximum columns in an index: 32
  Maximum size of a TOAST chunk: 1996
  Size of a large-object chunk: 2048
  Date/time type storage: 64-bit integers
  Float8 argument passing: by value
  Data page checksum version: 1
  Mock authentication nonce: f6dfe4464252d43f17386483dfdaf87961a5633d17ee3386e6e60a1504df09e8

2024-12-26 23:43:02,198 INFO: Lock owner: patroni1; I am patroni2
2024-12-26 23:43:02,198 INFO: starting as a secondary
2024-12-26 23:43:02,699 INFO: postmaster pid=5170
2024-12-26 23:43:02,848 INFO: establishing a new patroni heartbeat connection to postgres
2024-12-26 23:43:02,876 INFO: establishing a new patroni heartbeat connection to postgres
2024-12-26 23:43:03,770 INFO: Lock owner: patroni1; I am patroni2
2024-12-26 23:43:03,770 INFO: establishing a new patroni heartbeat connection to postgres
2024-12-26 23:43:03,902 INFO: no action. I am (patroni2), a secondary, and following a leader (patroni1)
2024-12-26 23:43:04,694 INFO: establishing a new patroni restapi connection to postgres
2024-12-26 23:43:09,718 INFO: no action. I am (patroni2), a secondary, and following a leader (patroni1)
2024-12-26 23:43:20,260 INFO: no action. I am (patroni2), a secondary, and following a leader (patroni1)

```

5.3. Попробуем "убить" процесс patroni na мастер ноде:

Посмотрим, pid процесса на мастере

```
daa@patroni1:~$ sudo systemctl status patroni
[sudo] password for daa:
● patroni.service - Runners to orchestrate a high-availability PostgreSQL
     Loaded: loaded (/usr/lib/systemd/system/patroni.service; enabled; preset: enabled)
     Active: active (running) since Wed 2024-12-25 23:17:57 MSK; 1min 31s ago
   Main PID: 75738 (patroni)
      Tasks: 17 (limit: 4558)
     Memory: 2.0G (peak: 2.0G)
        CPU: 2.236s
     CGroup: /system.slice/patroni.service
             ├─71744 /usr/lib/postgresql/15/bin/postgres -D /var/lib/postgresql/15/main --config-file=/etc/postgresql/15/main/p>
             ├─71746 "postgres: patroni-cluster: logger "
             ├─71747 "postgres: patroni-cluster: checkpointer "
             ├─71748 "postgres: patroni-cluster: background writer "
             ├─72980 "postgres: patroni-cluster: walwriter "
             ├─72981 "postgres: patroni-cluster: autovacuum launcher "
             ├─72982 "postgres: patroni-cluster: archiver last was 0000002D00000000000000A9.partial"
             ├─72983 "postgres: patroni-cluster: logical replication launcher "
             ├─74374 "postgres: patroni-cluster: walsender replicator 192.168.1.112(60794) streaming 0/A90002D0"
             ├─75738 /usr/bin/python3 /usr/bin/patroni /etc/patroni/config.yml
             ├─75746 "postgres: patroni-cluster: postgres postgres [local] idle"
             └─75757 "postgres: patroni-cluster: postgres postgres [local] idle"
```

"Убъем" процесс по pid:

```
daa@patroni1:~$ sudo kill -9 75738
```

Посмотрим в логи patroni на мастере:

```
2024-12-25 23:17:54,609 INFO: no action. I am (patroni1), the leader with the lock
2024-12-25 23:17:57,893 INFO: Selected new etcd server http://192.168.1.109:2379
2024-12-25 23:17:57,896 WARNING: I am the leader but not owner of the lease
2024-12-25 23:17:57,902 INFO: No PostgreSQL configuration items changed, nothing to reload.
2024-12-25 23:17:57,969 INFO: establishing a new patroni heartbeat connection to postgres
2024-12-25 23:17:57,995 INFO: Changed archive_mode from 'on' to 'True' (restart might be required)
2024-12-25 23:17:57,995 INFO: Changed synchronous_commit from 'on' to 'True'
2024-12-25 23:17:57,996 INFO: Changed wal_compression from 'pglz' to 'True'
2024-12-25 23:17:58,004 INFO: Reloading PostgreSQL configuration.
2024-12-25 23:17:59,529 WARNING: I am the leader but not owner of the lease
2024-12-25 23:17:59,751 INFO: no action. I am (patroni1), the leader with the lock
2024-12-25 23:17:59,870 INFO: establishing a new patroni restapi connection to postgres
2024-12-25 23:18:09,615 INFO: no action. I am (patroni1), the leader with the lock
2024-12-25 23:18:19,572 INFO: no action. I am (patroni1), the leader with the lock
```

И на второй ноде:

```
2024-12-25 23:17:07,136 INFO: no action. I am (patroni2), a secondary, and following a leader (patroni1)
2024-12-25 23:17:17,179 INFO: no action. I am (patroni2), a secondary, and following a leader (patroni1)
2024-12-25 23:17:27,136 INFO: no action. I am (patroni2), a secondary, and following a leader (patroni1)
2024-12-25 23:17:37,179 INFO: no action. I am (patroni2), a secondary, and following a leader (patroni1)
2024-12-25 23:17:47,136 INFO: no action. I am (patroni2), a secondary, and following a leader (patroni1)
2024-12-25 23:17:57,179 INFO: no action. I am (patroni2), a secondary, and following a leader (patroni1)
2024-12-25 23:17:59,721 INFO: no action. I am (patroni2), a secondary, and following a leader (patroni1)
2024-12-25 23:18:09,659 INFO: no action. I am (patroni2), a secondary, and following a leader (patroni1)
2024-12-25 23:18:20,159 INFO: no action. I am (patroni2), a secondary, and following a leader (patroni1)
```

Процесс был самостоятельно перезапущен, однако прерывание по операциям записи все таки зафиксировалось, но переключения не произошло

```
 Working with:   MASTER - 192.168.1.111
     Inserted: 2024-12-25 23:21:41.070177

Trying to connect
Unable to connect to database :
connection to server at "192.168.1.115", port 5000 failed: server closed the connection unexpectedly
        This probably means the server terminated abnormally
        before or while processing the request.

daa@test1:~$ ./HAtester.py 5000
Unable to connect to database :
connection to server at "192.168.1.115", port 5000 failed: server closed the connection unexpectedly
        This probably means the server terminated abnormally
        before or while processing the request.

daa@test1:~$ ./HAtester.py 5000
Unable to connect to database :
connection to server at "192.168.1.115", port 5000 failed: server closed the connection unexpectedly
        This probably means the server terminated abnormally
        before or while processing the request.

daa@test1:~$ ./HAtester.py 5000
Unable to connect to database :
connection to server at "192.168.1.115", port 5000 failed: server closed the connection unexpectedly
        This probably means the server terminated abnormally
        before or while processing the request.

daa@test1:~$ ./HAtester.py 5000
^[[A Working with:   MASTER - 192.168.1.111
     Inserted: 2024-12-25 23:21:48.588856
```

5.4. Отключение etcd:

```
daa@etcd1:~$ etcdctl -w table endpoint --cluster status
+---------------------------+------------------+---------+---------+-----------+------------+-----------+------------+--------------------+-----------------------+
|         ENDPOINT          |        ID        | VERSION | DB SIZE | IS LEADER | IS LEARNER | RAFT TERM | RAFT INDEX | RAFT APPLIED INDEX |        ERRORS         |
+---------------------------+------------------+---------+---------+-----------+------------+-----------+------------+--------------------+-----------------------+
| http://192.168.1.106:2379 | 2125f1467958dff8 |  3.5.17 |  127 kB |      true |      false |        17 |     413670 |             413670 |                       |
| http://192.168.1.110:2379 | 265b9c61ffe85ba1 |  3.5.17 |  139 kB |     false |      false |        17 |     413670 |             413670 |                       |
| http://192.168.1.109:2379 | 97ba8852ebaddf74 |  3.5.17 |  115 kB |     false |      false |        17 |     413645 |             413645 | etcdserver: no leader |
+---------------------------+------------------+---------+---------+-----------+------------+-----------+------------+--------------------+-----------------------+
```

Отключим сначала одну лидер ноду etcd1:

В логах patroni увидим ошибку взаимодействия с нодой, она периодически будет всплывать, однако кластер все еще работает

```
2024-12-27 00:00:49,802 INFO: no action. I am (patroni2), a secondary, and following a leader (patroni1)
2024-12-27 00:00:59,716 INFO: Lock owner: patroni1; I am patroni2
2024-12-27 00:01:01,387 ERROR: Failed to get list of machines from http://192.168.1.106:2379/v3: MaxRetryError("HTTPConnectionPool(host='192.168.1.106', port=2379): Max retries exceeded with url: /v3/cluster/member/list (Caused by ConnectTimeoutError(<urllib3.connection.HTTPConnection object at 0x7a542e3acec0>, 'Connection to 192.168.1.106 timed out. (connect timeout=1.6666666666666667)'))")
2024-12-27 00:01:01,445 INFO: no action. I am (patroni2), a secondary, and following a leader (patroni1)
```

Отключим еще одну ноду etcd3:

В логах patroni уже увидим полный отказ взаимодействия с etcd

```
2024-12-27 00:46:15,497 - ERROR - Request to server http://192.168.1.109:2379 failed: MaxRetryError("HTTPConnectionPool(host='192.168.1.109', port=2379): Max retries exceeded with url: /v3/kv/range (Caused by NewConnectionError('<urllib3.connection.HTTPConnection object at 0x752c7ca78cb0>: Failed to establish a new connection: [Errno 113] No route to host'))")
2024-12-27 00:46:16,072 - ERROR - Request to server http://192.168.1.106:2379 failed: MaxRetryError("HTTPConnectionPool(host='192.168.1.106', port=2379): Max retries exceeded with url: /v3/kv/range (Caused by NewConnectionError('<urllib3.connection.HTTPConnection object at 0x752c7ca78da0>: Failed to establish a new connection: [Errno 113] No route to host'))")
2024-12-27 00:46:19,410 - ERROR - Request to server http://192.168.1.110:2379 failed: ReadTimeoutError("HTTPConnectionPool(host='192.168.1.110', port=2379): Read timed out. (read timeout=3.332968132333311)")
2024-12-27 00:46:21,080 - ERROR - Failed to get list of machines from http://192.168.1.106:2379/v3: MaxRetryError("HTTPConnectionPool(host='192.168.1.106', port=2379): Max retries exceeded with url: /v3/cluster/member/list (Caused by ConnectTimeoutError(<urllib3.connection.HTTPConnection object at 0x752c7e7d3200>, 'Connection to 192.168.1.106 timed out. (connect timeout=1.6666666666666667)'))")
patroni.dcs.etcd3.Etcd3Error: Etcd is not responding properly
```

Haproxy говорит, что не видит рабочих мастер нод, однако видит реплики

![image](https://github.com/user-attachments/assets/2a5c259b-9663-4630-8a15-27f67f7ab845)

И действительно, мы можем обратиться к реплике в режиме чтения:

```

 Working with:    REPLICA - 192.168.1.112
     Retrived: 2024-12-27 00:45:52.234837

 Working with:    REPLICA - 192.168.1.112
     Retrived: 2024-12-27 00:45:52.234837

 Working with:    REPLICA - 192.168.1.112
     Retrived: 2024-12-27 00:45:52.234837

```

Попробуем подключиться к БД и внести изменения, к той ноде, что была репликой, до манипуляций с etcd

```
daa@patroni2:~$ psql -p 6432 -U postgres
Password for user postgres:
psql (15.10 (Ubuntu 15.10-1.pgdg24.04+1))
Type "help" for help.

postgres=# \c demo
You are now connected to database "demo" as user "postgres".
demo=# CREATE TABLE splitblain(c1 text);
ERROR:  cannot execute CREATE TABLE in a read-only transaction
demo=#
```

Получим ошибку, которая говорит, что в транзакциях только для чтения, нельзя вносить изменения.

Попробуем подключиться к БД и внести изменения, к той ноде, что была мастером, до манипуляций с etcd

```
daa@patroni1:~$ psql -p 6432 -U postgres
Password for user postgres:
psql (15.10 (Ubuntu 15.10-1.pgdg24.04+1))
Type "help" for help.

postgres=# \c demo
You are now connected to database "demo" as user "postgres".
demo=# CREATE TABLE splitblain(c1 text);
ERROR:  cannot execute CREATE TABLE in a read-only transaction
demo=#
```

Получим аналогичную ошибку. Таким образом из БД можно прочитать данные, но занести нельзя.

Теперь восстановим etcd1 и 3:

patroni восстановился

```
2024-12-27 00:56:42,800 ERROR: Failed to get list of machines from http://192.168.1.109:2379/v3: MaxRetryError("HTTPConnectionPool(host='192.168.1.109', port=2379): Max retries exceeded with url: /v3/cluster/member/list (Caused by NewConnectionError('<urllib3.connection.HTTPConnection object at 0x7cd71e327f80>: Failed to establish a new connection: [Errno 113] No route to host'))")
2024-12-27 00:56:42,804 INFO: Lock owner: patroni1; I am patroni1
2024-12-27 00:56:42,850 WARNING: Dropping physical replication slot patroni2 because of its xmin value 32600
2024-12-27 00:56:42,850 WARNING: Unable to drop replication slot 'patroni2', slot is active
2024-12-27 00:56:42,893 INFO: promoted self to leader because I had the session lock
2024-12-27 00:56:44,032 INFO: no action. I am (patroni1), the leader with the lock
2024-12-27 00:56:53,942 INFO: no action. I am (patroni1), the leader with the lock
```

```
2024-12-27 00:56:38,340 INFO: no action. I am (patroni2), a secondary, and following a leader (patroni1)
2024-12-27 00:56:38,342 WARNING: Loop time exceeded, rescheduling immediately.
2024-12-27 00:56:38,382 INFO: Lock owner: patroni1; I am patroni2
2024-12-27 00:56:41,721 ERROR: Request to server http://192.168.1.110:2379 failed: ReadTimeoutError("HTTPConnectionPool(host='192.168.1.110', port=2379): Read timed out. (read timeout=3.3330457093328127)")
2024-12-27 00:56:41,721 INFO: Reconnection allowed, looking for another server.
2024-12-27 00:56:41,721 INFO: Retrying on http://192.168.1.109:2379
2024-12-27 00:56:45,058 ERROR: Request to server http://192.168.1.109:2379 failed: ReadTimeoutError("HTTPConnectionPool(host='192.168.1.109', port=2379): Read timed out. (read timeout=1.7717137793327615)")
2024-12-27 00:56:45,058 INFO: Reconnection allowed, looking for another server.
2024-12-27 00:56:45,058 INFO: Retrying on http://192.168.1.106:2379
2024-12-27 00:56:45,060 INFO: Selected new etcd server http://192.168.1.106:2379
2024-12-27 00:56:45,061 ERROR: watchprefix failed: ProtocolError("Connection broken: InvalidChunkLength(got length b'', 0 bytes read)", InvalidChunkLength(got length b'', 0 bytes read))
2024-12-27 00:56:45,146 INFO: no action. I am (patroni2), a secondary, and following a leader (patroni1)
2024-12-27 00:56:45,190 INFO: no action. I am (patroni2), a secondary, and following a leader (patroni1)
2024-12-27 00:56:55,690 INFO: no action. I am (patroni2), a secondary, and following a leader (patroni1)
```

```
daa@patroni1:~$ sudo patronictl -c /etc/patroni/config.yml list
[sudo] password for daa:
+ Cluster: patroni-cluster (7449702151863418585) +----+-----------+
| Member   | Host          | Role    | State     | TL | Lag in MB |
+----------+---------------+---------+-----------+----+-----------+
| patroni1 | 192.168.1.111 | Leader  | running   | 53 |           |
| patroni2 | 192.168.1.112 | Replica | streaming | 53 |         0 |
+----------+---------------+---------+-----------+----+-----------+
```

![image](https://github.com/user-attachments/assets/30e9cc85-6122-40a6-9840-39b5ecd87398)




