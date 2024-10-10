
### Установка и настройка PostgreSQL

Цель:
- создавать дополнительный диск для уже существующей виртуальной машины, размечать его и делать на нем файловую систему
- переносить содержимое базы данных PostgreSQL на дополнительный диск
- переносить содержимое БД PostgreSQL между виртуальными машинами

#### Cоздать новый проект в Яндекс облако или на любых ВМ, докере.

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

#### Попробуем установить postgresql 15. Получим ошибку

```
daa@daa-VMware-Virtual-Platform:~$ sudo apt install postgresql-client-15 postgresql-15
Чтение списков пакетов… Готово
Построение дерева зависимостей… Готово
Чтение информации о состоянии… Готово
E: Невозможно найти пакет postgresql-client-15
E: Невозможно найти пакет postgresql-15
```

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

#### Запустим psql из под УЗ postgres

```
daa@daa-VMware-Virtual-Platform:~$ sudo -u postgres psql
psql (17.0 (Ubuntu 17.0-1.pgdg24.04+1), сервер 15.8 (Ubuntu 15.8-1.pgdg24.04+1))
Введите "help", чтобы получить справку.

postgres=#
```

#### Создадим бд и произвольную таблицу в ней

```
postgres=# CREATE DATABASE hw3;
CREATE DATABASE
postgres=# \c hw3
psql (17.0 (Ubuntu 17.0-1.pgdg24.04+1), сервер 15.8 (Ubuntu 15.8-1.pgdg24.04+1))
Вы подключены к базе данных "hw3" как пользователь "postgres".
hw3=# create table test(c1 text);
CREATE TABLE
hw3=# insert into test values('1');
INSERT 0 1
hw3=# \q
```

#### Остановим Postgres через sudo -u postgres pg_ctlcluster 15 main stop

```
daa@daa-VMware-Virtual-Platform:~$ sudo -u postgres pg_ctlcluster 15 main stop
Warning: stopping the cluster using pg_ctlcluster will mark the systemd unit as failed. Consider using systemctl:
  sudo systemctl stop postgresql@15-main
daa@daa-VMware-Virtual-Platform:~$ sudo -u postgres pg_lsclusters
Ver Cluster Port Status Owner    Data directory              Log file
15  main    5432 down   postgres /var/lib/postgresql/15/main /var/log/postgresql/postgresql-15-main.log
```

#### Добавим диск 10Гб к ВМ

![image](https://github.com/user-attachments/assets/72605749-85d0-4820-a971-4752e28e10fa)


#### Идентифицируем диск в системе

```
daa@daa-VMware-Virtual-Platform:~$ sudo parted -l | grep Error
[sudo] пароль для daa:
Ошибка: /dev/sdb: метка диска не определена
```

#### Разметим диск в GPT

```
daa@daa-VMware-Virtual-Platform:~$ sudo parted /dev/sdb mklabel gpt
```

#### Создадим раздел

```
daa@daa-VMware-Virtual-Platform:~$ sudo parted -a opt /dev/sdb mkpart primary ext4 0% 100%
```

#### Посмотрим на него

```
daa@daa-VMware-Virtual-Platform:~$ lsblk
NAME   MAJ:MIN RM   SIZE RO TYPE MOUNTPOINTS
fd0      2:0    1     4K  0 disk
loop0    7:0    0     4K  1 loop /snap/bare/5
loop1    7:1    0 269,8M  1 loop /snap/firefox/4793
loop2    7:2    0  74,2M  1 loop /snap/core22/1621
loop3    7:3    0  74,3M  1 loop /snap/core22/1612
loop4    7:4    0 505,1M  1 loop /snap/gnome-42-2204/176
loop5    7:5    0  10,7M  1 loop /snap/firmware-updater/127
loop6    7:6    0  91,7M  1 loop /snap/gtk-common-themes/1535
loop7    7:7    0  10,5M  1 loop /snap/snap-store/1173
loop8    7:8    0  38,8M  1 loop /snap/snapd/21759
loop9    7:9    0   564K  1 loop /snap/snapd-desktop-integration/247
loop10   7:10   0   500K  1 loop /snap/snapd-desktop-integration/178
sda      8:0    0    20G  0 disk
├─sda1   8:1    0     1M  0 part
└─sda2   8:2    0    20G  0 part /
**sdb      8:16   0    10G  0 disk
└─sdb1   8:17   0    10G  0 part**
sr0     11:0    1    88M  0 rom  /media/daa/CDROM
sr1     11:1    1   5,8G  0 rom  /media/daa/Ubuntu 24.04.1 LTS amd64
```

#### Создадим файловую системы

```
daa@daa-VMware-Virtual-Platform:~$ sudo mkfs.ext4 -L datapartition /dev/sdb1
mke2fs 1.47.0 (5-Feb-2023)
Creating filesystem with 2620928 4k blocks and 655360 inodes
Filesystem UUID: 2e01e74a-eb0d-40c6-b911-20ddc205370a
Superblock backups stored on blocks:
        32768, 98304, 163840, 229376, 294912, 819200, 884736, 1605632

Allocating group tables: done
Сохранение таблицы inod'ов: done
Создание журнала (16384 блоков): готово
Writing superblocks and filesystem accounting information: готово
```

#### Добавим в fstab для автоматического монтирования при запуске системы

```
daa@daa-VMware-Virtual-Platform:~$ sudo nano /etc/fstab
```

![image](https://github.com/user-attachments/assets/8d10ab48-443e-412d-b8e6-14717a6e916c)

#### Перезагрузим ВМ и проверим результат

```
daa@daa-VMware-Virtual-Platform:~$ sudo reboot now

Broadcast message from root@daa-VMware-Virtual-Platform on pts/1 (Wed 2024-10-09 16:23:06 MSK):

The system will reboot now!
```

```
Last login: Wed Oct  9 16:07:43 2024 from 192.168.11.1
daa@daa-VMware-Virtual-Platform:~$ lsblk
NAME   MAJ:MIN RM   SIZE RO TYPE MOUNTPOINTS
fd0      2:0    1     4K  0 disk
loop0    7:0    0     4K  1 loop /snap/bare/5
loop1    7:1    0  74,3M  1 loop /snap/core22/1612
loop2    7:2    0  74,2M  1 loop /snap/core22/1621
loop3    7:3    0 269,8M  1 loop /snap/firefox/4793
loop4    7:4    0  10,7M  1 loop /snap/firmware-updater/127
loop5    7:5    0 505,1M  1 loop /snap/gnome-42-2204/176
loop6    7:6    0  91,7M  1 loop /snap/gtk-common-themes/1535
loop7    7:7    0  38,8M  1 loop /snap/snapd/21759
loop8    7:8    0  10,5M  1 loop /snap/snap-store/1173
loop9    7:9    0   500K  1 loop /snap/snapd-desktop-integration/178
loop10   7:10   0   564K  1 loop /snap/snapd-desktop-integration/247
sda      8:0    0    20G  0 disk
├─sda1   8:1    0     1M  0 part
└─sda2   8:2    0    20G  0 part /
**sdb      8:16   0    10G  0 disk
└─sdb1   8:17   0    10G  0 part /mnt/data**
sr0     11:0    1    88M  0 rom
sr1     11:1    1   5,8G  0 rom
```

#### Сделаем пользователя postgres владельцем /mnt/data

```
daa@daa-VMware-Virtual-Platform:~$ sudo chown -R postgres:postgres /mnt/data
daa@daa-VMware-Virtual-Platform:~$ ls -l /mnt/data
итого 16
drwx------ 2 postgres postgres 16384 окт  9 16:00 lost+found
```

#### Перенесем все из папки var/lib/postgres/15 в /mnt/data

```
daa@daa-VMware-Virtual-Platform:~$ sudo mv /var/lib/postgresql/15 /mnt/data
```

#### Попробуем запустить кластер и получим ошибку. Ошибка связана с тем, что файлы БД были перемещены на прошлом этапе

```
daa@daa-VMware-Virtual-Platform:~$ sudo -u postgres pg_ctlcluster 15 main start
Error: /var/lib/postgresql/15/main is not accessible or does not exist
```

#### Поменяем конфигурационный файл postgresql.conf, пропишем корректный путь

```
daa@daa-VMware-Virtual-Platform:~$ cd /etc/postgresql/15/main
daa@daa-VMware-Virtual-Platform:/etc/postgresql/15/main$ sudo nano postgresql.conf
```
![image](https://github.com/user-attachments/assets/0bceaff1-98fb-4c06-be9e-5913cb1e36bd)

#### Теперь кластер успешно запускается

```
daa@daa-VMware-Virtual-Platform:/etc/postgresql/15/main$ sudo -u postgres pg_ctlcluster 15 main start
Warning: the cluster will not be running as a systemd service. Consider using systemctl:
  sudo systemctl start postgresql@15-main
daa@daa-VMware-Virtual-Platform:/etc/postgresql/15/main$ sudo -u postgres pg_lsclusters
Ver Cluster Port Status Owner    Data directory    Log file
15  main    5432 online postgres /mnt/data/15/main /var/log/postgresql/postgresql-15-main.log
```

#### Проверим, что там в БД. Таблица на месте

```
daa@daa-VMware-Virtual-Platform:/etc/postgresql/15/main$ sudo -u postgres psql
psql (17.0 (Ubuntu 17.0-1.pgdg24.04+1), сервер 15.8 (Ubuntu 15.8-1.pgdg24.04+1))
Введите "help", чтобы получить справку.

postgres=# \c hw3
psql (17.0 (Ubuntu 17.0-1.pgdg24.04+1), сервер 15.8 (Ubuntu 15.8-1.pgdg24.04+1))
Вы подключены к базе данных "hw3" как пользователь "postgres".
hw3=# select * from test;
 c1
----
 1
(1 строка)
```


### Задача со *

не удаляя существующий инстанс ВМ сделайте новый, поставьте на его PostgreSQL, удалите файлы с данными из /var/lib/postgres, перемонтируйте внешний диск который сделали ранее от первой виртуальной машины ко второй и запустите PostgreSQL на второй машине так чтобы он работал с данными на внешнем диске, расскажите как вы это сделали и что в итоге получилось.

#### Подготовим новую ВМ

Создадим новую ВМ в VMWare Workstation.
Включим SSH, установим PostgreSQL15 и подключимся через Putty

Проверим кластер после установки
```
daa2@daa2-VMware-Virtual-Platform:~$ sudo -u postgres pg_lsclusters
Ver Cluster Port Status Owner    Data directory              Log file
15  main    5432 online postgres /var/lib/postgresql/15/main /var/log/postgresql/postgresql-15-main.log
```

#### Остановим кластер

```
daa2@daa2-VMware-Virtual-Platform:~$ sudo -u postgres pg_ctlcluster 15 main stop
Warning: stopping the cluster using pg_ctlcluster will mark the systemd unit as failed. Consider using systemctl:
  sudo systemctl stop postgresql@15-main
daa2@daa2-VMware-Virtual-Platform:~$ sudo -u postgres pg_lsclusters
Ver Cluster Port Status Owner    Data directory              Log file
15  main    5432 down   postgres /var/lib/postgresql/15/main /var/log/postgresql/postgresql-15-main.log
```

#### Удалим содержимое /var/lib/postgres

```
daa2@daa2-VMware-Virtual-Platform:/var/lib/postgresql$ ls
15
daa2@daa2-VMware-Virtual-Platform:/var/lib/postgresql$ sudo rm -r /var/lib/postgresql
daa2@daa2-VMware-Virtual-Platform:/var/lib/postgresql$ ls
```

#### Примонтируем дистк от первой ВМ ко второй

![image](https://github.com/user-attachments/assets/751ce2ab-b405-436b-ae33-6273f069d2c8)

#### Проверим результат. ОС видит диск

```
daa2@daa2-VMware-Virtual-Platform:~$ lsblk
NAME   MAJ:MIN RM   SIZE RO TYPE MOUNTPOINTS
fd0      2:0    1     4K  0 disk
loop0    7:0    0     4K  1 loop /snap/bare/5
loop1    7:1    0  74.3M  1 loop /snap/core22/1564
loop2    7:2    0 269.8M  1 loop /snap/firefox/4793
loop3    7:3    0  10.7M  1 loop /snap/firmware-updater/127
loop4    7:4    0  38.8M  1 loop /snap/snapd/21759
loop5    7:5    0  91.7M  1 loop /snap/gtk-common-themes/1535
loop6    7:6    0  10.5M  1 loop /snap/snap-store/1173
loop7    7:7    0 505.1M  1 loop /snap/gnome-42-2204/176
loop8    7:8    0   500K  1 loop /snap/snapd-desktop-integration/178
sda      8:0    0    20G  0 disk
├─sda1   8:1    0     1M  0 part
└─sda2   8:2    0    20G  0 part /
**sdb      8:16   0    10G  0 disk
└─sdb1   8:17   0    10G  0 part**
sr0     11:0    1    88M  0 rom
sr1     11:1    1   5.8G  0 rom
```

#### Установим parted

```
daa2@daa2-VMware-Virtual-Platform:~$ sudo apt install parted
Reading package lists... Done
Building dependency tree... Done
Reading state information... Done
parted is already the newest version (3.6-4build1).
parted set to manually installed.
0 upgraded, 0 newly installed, 0 to remove and 34 not upgraded.
```

#### Создадим дирректорию /mnt/data, примонтируем диск к ней и дадим прав пользователю postgre

```
daa2@daa2-VMware-Virtual-Platform:~$ sudo mkdir -p /mnt/data
daa2@daa2-VMware-Virtual-Platform:~$ sudo mount -o defaults /dev/sdb1 /mnt/data
daa2@daa2-VMware-Virtual-Platform:~$ sudo chown -R postgres:postgres /mnt/data
```
```
daa2@daa2-VMware-Virtual-Platform:~$ lsblk
NAME   MAJ:MIN RM   SIZE RO TYPE MOUNTPOINTS
fd0      2:0    1     4K  0 disk
loop0    7:0    0     4K  1 loop /snap/bare/5
loop1    7:1    0  74.3M  1 loop /snap/core22/1564
loop2    7:2    0 269.8M  1 loop /snap/firefox/4793
loop3    7:3    0  10.7M  1 loop /snap/firmware-updater/127
loop4    7:4    0  38.8M  1 loop /snap/snapd/21759
loop5    7:5    0  91.7M  1 loop /snap/gtk-common-themes/1535
loop6    7:6    0  10.5M  1 loop /snap/snap-store/1173
loop7    7:7    0 505.1M  1 loop /snap/gnome-42-2204/176
loop8    7:8    0   500K  1 loop /snap/snapd-desktop-integration/178
sda      8:0    0    20G  0 disk
├─sda1   8:1    0     1M  0 part
└─sda2   8:2    0    20G  0 part /
**sdb      8:16   0    10G  0 disk
└─sdb1   8:17   0    10G  0 part /mnt/data**
sr0     11:0    1    88M  0 rom
sr1     11:1    1   5.8G  0 rom
```


#### Добавим в файле конфирурации соответствующую строчку, чтобы работало автомонтирование при запуске системы

```
daa2@daa2-VMware-Virtual-Platform:~$ sudo nano /etc/fstab 
```
![image](https://github.com/user-attachments/assets/f0832306-e528-4fd9-8004-012d151d845d)

#### Изменим конфигурацию PostgreSQL

```
daa2@daa2-VMware-Virtual-Platform:~$ cd /etc/postgresql/15/main
daa2@daa2-VMware-Virtual-Platform:/etc/postgresql/15/main$ sudo nano postgresql.conf
```

![image](https://github.com/user-attachments/assets/ede11734-f6b3-4d10-a73d-0a982523a98f)

#### Попробуем запустить кластер

```
daa2@daa2-VMware-Virtual-Platform:/etc/postgresql/15/main$ sudo -u postgres pg_ctlcluster 15 main start
Warning: the cluster will not be running as a systemd service. Consider using systemctl:
  sudo systemctl start postgresql@15-main
Warning: connection to the database failed, disabling startup checks:
psql: error: connection to server on socket "/var/run/postgresql/.s.PGSQL.5432" failed: FATAL:  database locale is incompatible with operating system
DETAIL:  The database was initialized with LC_COLLATE "ru_RU.UTF-8",  which is not recognized by setlocale().
```
Получили ошибку кодировки, т.к. БД была инициализирована в кодировке ru_RU.UTF-8

#### Установим необходимую кодировку 

```
daa2@daa2-VMware-Virtual-Platform:/etc/postgresql/15/main$ sudo locale-gen ru_RU.UTF-8
Generating locales (this might take a while)...
  ru_RU.UTF-8... done
Generation complete.
daa2@daa2-VMware-Virtual-Platform:/etc/postgresql/15/main$
Broadcast message from daa2@daa2-VMware-Virtual-Platform (Thu 2024-10-10 13:47:00 MSK):

The system will reboot now!
```

#### После ребута кластер запустился сам

```
daa2@daa2-VMware-Virtual-Platform:~$ sudo -u postgres pg_lsclusters
Ver Cluster Port Status Owner    Data directory    Log file
15  main    5432 online postgres /mnt/data/15/main /var/log/postgresql/postgresql-15-main.log
```

#### Подключимся к БД и проверим данные

```
daa2@daa2-VMware-Virtual-Platform:/etc/postgresql/15/main$ sudo -u postgres psql
psql (15.8 (Ubuntu 15.8-1.pgdg24.04+1))
Type "help" for help.

postgres=# \c hw3
You are now connected to database "hw3" as user "postgres".
hw3=# select *from test;
 c1
----
 1
(1 row)
```

Данные на месте, но кажется, что я просто не столкнулся с теми проблемами на которые был расчет при выполнении этого задания. Возможно это связано с тем, что использовалось одно и то же ядро убунту.
