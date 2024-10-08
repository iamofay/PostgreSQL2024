### Установка и настройка PostgteSQL в контейнере Docker

Цель:
- установить PostgreSQL в Docker контейнере
- настроить контейнер для внешнего подключения

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

### Поставим Doker Engine в соответствии с инструкцией на сайте вендора

```
https://docs.docker.com/engine/install/ubuntu/#installation-methods
```

#### Проверим что Doker установлен

```
daa@daa-VMware-Virtual-Platform:~$  sudo docker run hello-world
[sudo] пароль для daa:

Hello from Docker!
This message shows that your installation appears to be working correctly.

To generate this message, Docker took the following steps:
 1. The Docker client contacted the Docker daemon.
 2. The Docker daemon pulled the "hello-world" image from the Docker Hub.
    (amd64)
 3. The Docker daemon created a new container from that image which runs the
    executable that produces the output you are currently reading.
 4. The Docker daemon streamed that output to the Docker client, which sent it
    to your terminal.

To try something more ambitious, you can run an Ubuntu container with:
 $ docker run -it ubuntu bash

Share images, automate workflows, and more with a free Docker ID:
 https://hub.docker.com/

For more examples and ideas, visit:
 https://docs.docker.com/get-started/
```

#### Создадим подсеть для докера

```
daa@daa-VMware-Virtual-Platform:~$ sudo docker network create pg-net
63848dd8157fcd9882caa0adff1e54264733f840aef3fe7e4e3136ddc0dc9ea6
```

#### Создадим контейнер с PostgreSQL 15 и смонтируем в него каталог /var/lib/postgresql

```
daa@daa-VMware-Virtual-Platform:~$ sudo docker run --name pg-server --network pg-net -e POSTGRES_PASSWORD=postgres -d -p 5432:5432 -v /var/lib/postgres:/var/lib/postgresql/data postgres:15
f34d76e983befa43965b8b3bb5b0b4b5e276f520b53b99ab4d3f0480161ea667
```

#### Создадим контейнер с клиентом PostgreSQL и подключимся

```
daa@daa-VMware-Virtual-Platform:~$ sudo docker run -it --network pg-net --name pg-client postgres:15 psql -h pg-server -U postgres
Password for user postgres:
psql (15.8 (Debian 15.8-1.pgdg120+1))
Type "help" for help.

postgres=#
```

#### Создадим БД и таблицу с пятью записями

```
postgres=# CREATE DATABASE hw2;
CREATE DATABASE
postgres-# \c hw2
You are now connected to database "hw2" as user "postgres".
hw2=# CREATE TABLE persons(id serial, first_name text, second_name text);
CREATE TABLE
hw2=# INSERT INTO persons(first_name, second_name) VALUES('petr','petrov');INSERT INTO persons(first_name, second_name) VALUES('ivan','ivanov');INSERT INTO persons(first_name, second_name) VALUES('dmitriy','dmitrov');INSERT INTO persons(first_name, second_name) VALUES('sergey','sergeev');INSERT INTO persons(first_name, second_name) VALUES('petr','petrov');
INSERT 0 1
INSERT 0 1
INSERT 0 1
INSERT 0 1
INSERT 0 1
hw2=# select * from persons                                                                                                   ;
 id | first_name | second_name
----+------------+-------------
  1 | petr       | petrov
  2 | ivan       | ivanov
  3 | dmitriy    | dmitrov
  4 | sergey     | sergeev
  5 | petr       | petrov
(5 rows)

```

#### Убедимся, что мы заходили под клиентским контейнером

```
daa@daa-VMware-Virtual-Platform:~$ sudo docker ps -a
CONTAINER ID   IMAGE         COMMAND                  CREATED          STATUS                      PORTS                                       NAMES
4853e37904a2   postgres:15   "docker-entrypoint.s…"   54 minutes ago   Exited (0) 43 minutes ago                                               pg-client
f34d76e983be   postgres:15   "docker-entrypoint.s…"   55 minutes ago   Up 55 minutes               0.0.0.0:5432->5432/tcp, :::5432->5432/tcp   pg-server
fd8e12f925ea   hello-world   "/hello"                 25 hours ago     Exited (0) 25 hours ago                                                 confident_bhaskara
26da429fa6fb   hello-world   "/hello"                 26 hours ago     Exited (0) 26 hours ago                                                 sharp_lamarr
```

#### Подключимся к БД с помощью DBeaver извне

![image](https://github.com/user-attachments/assets/c8a5a8dc-80a3-4643-baa4-502802e4cc10)

![image](https://github.com/user-attachments/assets/2059474b-b676-41a2-b86b-f21da6f326bf)

#### Удалим контейнер с сервером БД

```
daa@daa-VMware-Virtual-Platform:~$ sudo docker ps -a
[sudo] пароль для daa:
CONTAINER ID   IMAGE         COMMAND                  CREATED             STATUS                         PORTS                                       NAMES
4853e37904a2   postgres:15   "docker-entrypoint.s…"   About an hour ago   Exited (0) About an hour ago                                               pg-client
f34d76e983be   postgres:15   "docker-entrypoint.s…"   About an hour ago   Up About an hour               0.0.0.0:5432->5432/tcp, :::5432->5432/tcp   pg-server
fd8e12f925ea   hello-world   "/hello"                 25 hours ago        Exited (0) 25 hours ago                                                    confident_bhaskara
26da429fa6fb   hello-world   "/hello"                 27 hours ago        Exited (0) 27 hours ago                                                    sharp_lamarr
daa@daa-VMware-Virtual-Platform:~$ sudo docker rm pg-server
Error response from daemon: cannot remove container "/pg-server": container is running: stop the container before removing or force remove
daa@daa-VMware-Virtual-Platform:~$ sudo docker stop pg-server
pg-server
daa@daa-VMware-Virtual-Platform:~$ sudo docker rm pg-server
pg-server
daa@daa-VMware-Virtual-Platform:~$ sudo docker ps -a
CONTAINER ID   IMAGE         COMMAND                  CREATED             STATUS                         PORTS     NAMES
4853e37904a2   postgres:15   "docker-entrypoint.s…"   About an hour ago   Exited (0) About an hour ago             pg-client
fd8e12f925ea   hello-world   "/hello"                 25 hours ago        Exited (0) 25 hours ago                  confident_bhaskara
26da429fa6fb   hello-world   "/hello"                 27 hours ago        Exited (0) 27 hours ago                  sharp_lamarr
```

#### Создадим заново

```
daa@daa-VMware-Virtual-Platform:~$ sudo docker run --name pg-server --network pg-net -e POSTGRES_PASSWORD=postgres -d -p 5432:5432 -v /var/lib/postgres:/var/lib/postgresql/data postgres:15
42dcfd18103a23a34c30eeb3397c101b5edaf11a3d686eb3bb372efe11e0ce65
```

#### Подключимся из контейнера с клиентом к контейнеру с сервером и увидим, что данные остались на месте

```
daa@daa-VMware-Virtual-Platform:~$ sudo docker run -it --network pg-net --name pg-client postgres:15 psql -h pg-server -U postgres
Password for user postgres:
psql (15.8 (Debian 15.8-1.pgdg120+1))
Type "help" for help.

postgres=# \c hw2
You are now connected to database "hw2" as user "postgres".
hw2=# select * from persons
hw2-# ;
 id | first_name | second_name
----+------------+-------------
  1 | petr       | petrov
  2 | ivan       | ivanov
  3 | dmitriy    | dmitrov
  4 | sergey     | sergeev
  5 | petr       | petrov
(5 rows)
```


