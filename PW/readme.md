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
ETCD_NAME=etcd1
##ETCD_DATA_DIR=/var/lib/etcd
ETCD_INITIAL_CLUSTER=etcd1=http://192.168.1.106:2380,etcd2=http://192.168.1.109:2380,etcd3=http://192.168.1.110:2380
ETCD_INITIAL_ADVERTISE_PEER_URLS=http://192.168.1.106:2380
ETCD_ADVERTISE_CLIENT_URLS=http://192.168.1.106:2379
ETCD_LISTEN_PEER_URLS=http://0.0.0.0:2380
ETCD_LISTEN_CLIENT_URLS=http://0.0.0.0:2379
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

daa@etcd1:~$ etcdctl --endpoints=http://192.168.1.106:2379 member list
3bbfd6bd0acc17c6, started, etcd2, http://192.168.1.109:2380, http://192.168.1.109:2379, false
559fadbf2c74b9f0, started, etcd3, http://192.168.1.110:2380, http://192.168.1.110:2379, false
73079f127bc11ea8, started, etcd1, http://192.168.1.106:2380, http://192.168.1.106:2379, false

