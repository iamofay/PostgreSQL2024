### Проектная работа. Создание и тестирование высоконагруженного отказоустойчивого кластера PostgreSQL на базе Patroni.

#### Подготовим кластер etcd

etcd1 192.168.1.106
etcd2 192.168.1.109
etcd3 192.168.1.110

![image](https://github.com/user-attachments/assets/02174db7-6a00-40d2-9936-710762d7f82a)
![image](https://github.com/user-attachments/assets/b3339d60-a147-487c-adaf-e53ca34ee0ce)

#### Установим etcd

https://github.com/etcd-io/etcd/releases/tag/v3.5.17

   Установим версию v3.5.17

   ETCD_VER=v3.5.17

# choose either URL
GOOGLE_URL=https://storage.googleapis.com/etcd
GITHUB_URL=https://github.com/etcd-io/etcd/releases/download
DOWNLOAD_URL=${GOOGLE_URL}

rm -f /tmp/etcd-${ETCD_VER}-linux-amd64.tar.gz
rm -rf /tmp/etcd-download-test && mkdir -p /tmp/etcd-download-test

curl -L ${DOWNLOAD_URL}/${ETCD_VER}/etcd-${ETCD_VER}-linux-amd64.tar.gz -o /tmp/etcd-${ETCD_VER}-linux-amd64.tar.gz
tar xzvf /tmp/etcd-${ETCD_VER}-linux-amd64.tar.gz -C /tmp/etcd-download-test --strip-components=1
rm -f /tmp/etcd-${ETCD_VER}-linux-amd64.tar.gz

/tmp/etcd-download-test/etcd --version
/tmp/etcd-download-test/etcdctl version
/tmp/etcd-download-test/etcdutl version

# start a local etcd server
/tmp/etcd-download-test/etcd

# write,read to etcd
/tmp/etcd-download-test/etcdctl --endpoints=localhost:2379 put foo bar
/tmp/etcd-download-test/etcdctl --endpoints=localhost:2379 get foo

Результаты установки:
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

####Переместите двоичные файлы в свой PATH:

sudo cp /tmp/etcd-download-test/etcd* /usr/local/bin
Теперь создайте служебный файл:

sudo vim /etc/systemd/system/etcd.service
Добавьте эти строки в файл:

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

####Настроим кластер:

Откройте файл для редактирования:

sudo nano /etc/default/etcd
Этот файл содержит все переменные среды для кластера. Нам нужно определить все переменные на всех трех узлах:

По аналогии на всех 3х нодах

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

После внесения изменений на всех трех узлах запустите или перезапустите службу etcd с помощью команд:

sudo systemctl daemon-reload
sudo systemctl enable --now etcd

##To Restart, Use:
sudo systemctl restart etcd

Убедимся, что служба работает на всех узлах:

$ systemctl status etcd

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

#### Postgresql+patroni

Установим postgre 

#### Утсановим некоторые пакеты

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

#### Импортируем PostgreSQL GPG key для верификации установочного пакета

```
daa@daa-VMware-Virtual-Platform:~$ curl -fSsL https://www.postgresql.org/media/keys/ACCC4CF8.asc | gpg --dearmor | sudo tee /usr/share/keyrings/postgresql.gpg > /dev/null
```

#### Импортируем APT репозиторий PostgreSQL и проапдейтим перечень репозиториев.

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

#### Устанавливаем PostgreSQL 15 и убедимся что кластер запущен

```
daa@daa-VMware-Virtual-Platform:~$ sudo -u postgres pg_lsclusters
Ver Cluster Port Status Owner    Data directory              Log file
15  main    5432 online postgres /var/lib/postgresql/15/main /var/log/postgresql/postgresql-15-main.log
```

Удалим каталог

daa@patroni1:~$ sudo systemctl stop postgresql
daa@patroni1:~$ sudo rm -fr /var/lib/postgresql/15/main

Устанавливаеем patroni

192.168.1.111 - patroni1
192.168.1.112 - patroni2

Установим python

Шаг 5: Установка Patroni
Установите Patroni и PIP на все узлы:
sudo apt-get install python3-pip python3-dev libpq-dev -y
sudo apt-get install patroni -y


sudo apt install python3-venv
Then create a virtual environment in your project directory like this:

 python3 -m venv .venv
Now activate your virtual environment by running:

 source .venv/bin/activate

 Установите зависимости для работы Patroni на все узлы:
pip3 install psycopg2-binary
pip3 install wheel
pip3 install python-etcd

(.venv) daa@patroni1:~$ deactivate

sudo apt update

sudo apt install python3-pip python3-dev libpq-dev -y

Шаг 5: Установка Patroni
Установите Patroni и PIP на все узлы:
sudo apt-get install python3-pip python3-dev libpq-dev -y
sudo apt-get install patroni -y

Установите зависимости для работы Patroni на все узлы:
pip3 install psycopg2-binary
pip3 install wheel
pip3 install python-etcd

(.venv) daa@patroni1:~$ deactivate

daa@patroni1:~$ sudo apt-get install patroni

```
daa@patroni1:~$ sudo nano /etc/patroni/config.yml
```

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
    - local all postgres peer
    - local all all peer
    - host all all 0.0.0.0/0 md5
    - host replication replicator localhost trust
    - host replication replicator 192.168.1.0/24 md5


postgresql:
  listen: 192.168.1.111,127.0.0.1:5432 # адрес узла, на котором находится этот файл
  connect_address: 192.168.1.111:5432 # адрес узла, на котором находится этот файл
  use_unix_socket: true
  data_dir: /var/lib/postgresql/15/main # каталог данных
  bin_dir: /usr/lib/postgresql/15/bin
  config_dir: /etc/postgresql/15/main
  pgpass: /var/lib/postgresql/.pgpass_patroni
  authentication:
    replication:
      username: replicator
      password: QWEasd123
    superuser:
      username: postgres
      password: QWEasd123
  parameters:
    unix_socket_directories: /var/run/postgresql
  pg_hba: # должен содержать адреса ВСЕХ машин, используемых в кластере
    - local all postgres peer
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
  mode: off # Allowed values: off, automatic, required
  device: /dev/watchdog
  safety_margin: 5

tags:
  nofailover: false
  noloadbalance: false
  clonefrom: false
  nosync: false

```

Перезапустим кластера patroni

```
daa@patroni1:~$ sudo systemctl restart patroni
```

Перезапустим

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
Кластер поднялся

```
daa@patroni1:/var/log/postgresql$ sudo patronictl -c /etc/patroni/config.yml list
+ Cluster: patroni-cluster (7449702151863418585) +----+-----------+
| Member   | Host          | Role    | State     | TL | Lag in MB |
+----------+---------------+---------+-----------+----+-----------+
| patroni1 | 192.168.1.111 | Leader  | running   |  5 |           |
| patroni2 | 192.168.1.112 | Replica | streaming |  5 |         0 |
+----------+---------------+---------+-----------+----+-----------+
```

Развернем демо БД

sudo psql -f demo_big.sql -U postgres




#### Установим pgbouncer на обе ноды

```
daa@patroni1:~$ sudo apt-get install pgbouncer -y
```

Переместите конфигурационный файл по умолчанию:
```
daa@patroni1:/etc$ sudo mv /etc/pgbouncer/pgbouncer.ini{,.original}
```

Создайте и откройте для редактирования новый конфигурационный файл:
```
sudo nano /etc/pgbouncer/pgbouncer.ini
```

```
  GNU nano 7.2                                      /etc/pgbouncer/pgbouncer.ini
[databases]
postgres = host=127.0.0.1 port=5432 dbname=demo
* = host=127.0.0.1 port=5432

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

Добавим пользователя 

```
daa@patroni1:/etc$  sudo nano /etc/pgbouncer/userlist.txt
```

```
 GNU nano 7.2                                       /etc/pgbouncer/userlist.txt
"postgres" "QWEasd123"
```

Перезапустите PGBouncer, выполнив команду:
sudo systemctl daemon-reload
sudo systemctl restart pgbouncer

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

Подключимся к БД

```
daa@patroni1:/etc$ psql -p 6432 -U postgres
Password for user postgres:
psql (15.10 (Ubuntu 15.10-1.pgdg24.04+1))
Type "help" for help.

postgres=# \c demo
You are now connected to database "demo" as user "postgres".
demo=# \q
```

#### Развернем haproxy

Настроим виртуальный ip для кластера haproxy

Изменим конфигурацию интерфейса на обеих нодах и добавим ip адрес 192.168.1.115 как кластерный к интерфейсам.

```
  GNU nano 7.2                   90-NM-14f59568-5076-387a-aef6-10adfcca2e26.yaml
network:
  version: 2
  ethernets:
    ens33:
      renderer: NetworkManager
      match: {}
      addresses:
      - "192.168.1.113/24"
      - "192.168.1.115/24"
      networkmanager:
        uuid: "14f59568-5076-387a-aef6-10adfcca2e26"
        name: "netplan-ens33"
        passthrough:
          connection.timestamp: "1734937357"
          ipv4.address1: "192.168.1.113/24,192.168.1.1"
          ipv4.method: "manual"
          ipv6.method: "disabled"
          ipv6.ip6-privacy: "-1"
          proxy._: ""

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

Посмотрим результат в веб интерфейсе

![image](https://github.com/user-attachments/assets/6062cb65-d9b3-48dd-a847-c5a34a9817b2)

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




