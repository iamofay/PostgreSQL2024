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


