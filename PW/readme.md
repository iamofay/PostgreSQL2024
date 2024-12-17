### Проектная работа. Создание и тестирование высоконагруженного отказоустойчивого кластера PostgreSQL на базе Patroni.

1. Подготовим кластер etcd

etcd1 192.168.1.106
etcd2 192.168.1.109
etcd3 192.168.1.110

![image](https://github.com/user-attachments/assets/02174db7-6a00-40d2-9936-710762d7f82a)
![image](https://github.com/user-attachments/assets/b3339d60-a147-487c-adaf-e53ca34ee0ce)

2. Установим etcd

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

Переместите двоичные файлы в свой PATH:

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
