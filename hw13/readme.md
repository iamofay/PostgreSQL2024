### Секционирование таблицы

Цель:
- научиться выполнять секционирование таблиц в PostgreSQL;
- повысить производительность запросов и упростив управление данными;

#### Подготовим ВМ

Для реализации домашнего задания использовалась VMWare Worstation, была поднята ВМ с ОС Ubuntu 20.04.1 desaktop + demo БД

1. Разрешим подключения к БД

```
daa@daa-pgsql:~$ echo "listen_addresses = '*'" | sudo tee -a /etc/postgresql/15/main/postgresql.conf
[sudo] password for daa:
listen_addresses = '*'
```

2. Разрешим подключение пользователей к базе, в т.ч. для физической репликации

```
daa@daa-pgsql:~$ echo "host all all 0.0.0.0/0 md5" | sudo tee -a /etc/postgresql/15/main/pg_hba.conf
host all all 0.0.0.0/0 md5
```

3. Перезапустим кластера 

```
sudo systemctl restart postgresql
```

4. Установим демо БД

https://postgrespro.ru/education/demodb

```
daa@daa-pgsql:~$ sudo psql -f demo_big.sql -U postgres
[sudo] password for daa:
Password for user postgres:
SET
SET
SET
...
ALTER TABLE
ALTER TABLE
ALTER DATABASE
ALTER DATABASE
```
