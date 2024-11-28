### Работа с индексами

Цель:
- знать и уметь применять основные виды индексов PostgreSQL
- строить и анализировать план выполнения запроса
- уметь оптимизировать запросы для с использованием индексов

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
daa@daa-pgsql:~$ sudo psql -f demo-small-20170815.sql -U postgres
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

Модель достаточно простая, но для решения ДЗ подходит.

![image](https://github.com/user-attachments/assets/244eb792-0ad1-4a24-8387-b3792adcf0d2)


#### Создать индекс к какой-либо из таблиц вашей БД

Возьмем таблицу ticket_flights, т.к. она самая большая. В ней уже есть индекс ticket_flights_pkey - по сути праймари кей таблицы.


Создать индексы на БД, которые ускорят доступ к данным.
В данном задании тренируются навыки:

определения узких мест
написания запросов для создания индекса
оптимизации
Необходимо:
Создать индекс к какой-либо из таблиц вашей БД
Прислать текстом результат команды explain,
в которой используется данный индекс
Реализовать индекс для полнотекстового поиска
Реализовать индекс на часть таблицы или индекс
на поле с функцией
Создать индекс на несколько полей
Написать комментарии к каждому из индексов
Описать что и как делали и с какими проблемами
столкнулись
