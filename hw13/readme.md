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
Password for user postgres:
SET
SET
SET
...
ALTER TABLE
ALTER TABLE
ALTER TABLE
REFRESH MATERIALIZED VIEW
```
```
postgres=# \l

                                                 List of databases
   Name    |  Owner   | Encoding |   Collate   |    Ctype    | ICU Locale | Locale Provider |   Access privileges
-----------+----------+----------+-------------+-------------+------------+-----------------+-----------------------
 demo      | postgres | UTF8     | en_US.UTF-8 | en_US.UTF-8 |            | libc            |
 postgres  | postgres | UTF8     | en_US.UTF-8 | en_US.UTF-8 |            | libc            |
 template0 | postgres | UTF8     | en_US.UTF-8 | en_US.UTF-8 |            | libc            | =c/postgres          +
           |          |          |             |             |            |                 | postgres=CTc/postgres
 template1 | postgres | UTF8     | en_US.UTF-8 | en_US.UTF-8 |            | libc            | =c/postgres          +
           |          |          |             |             |            |                 | postgres=CTc/postgres
(4 rows)
```

#### Анализ структуры данных

- Ознакомьтесь с таблицами базы данных, особенно с таблицами bookings, tickets, ticket_flights, flights, boarding_passes, seats, airports, aircrafts.
- Определите, какие данные в таблице bookings или других таблицах имеют логическую привязку к диапазонам, по которым можно провести секционирование (например, дата бронирования, рейсы).

![image](https://github.com/user-attachments/assets/73d06b90-7d6f-42b7-921b-23af441a7386)

bookings - секционирование можно провести по полям:

book_date - дата бронирования
total_amount - общая стоимость

tickets - секционирование можно провести по полям:

ticket_no - номер билета
passenger_id - id пассажира

ticket_flights - секционирование можно провести по полям:

ticket_no - номер билета
amount - стоимость

flights - секционирование можно провести по полям:

flight_id - id полета
scheduled_departure - Планируемое время вылета         
scheduled_arrival  - Планируемое время прибытия             
actual_departure - фактическое время вылета   
actual_arrival - фактическое время прибытия  

boarding_passes - секционирование можно провести по полям:

ticket_no - номер билета

seats, airports, aircrafts - таблицы маленькая, тут секционирование не нужно

#### Выбор таблицы для секционирования

- Основной акцент делается на секционировании таблицы bookings. Но вы можете выбрать и другие таблицы, если видите в этом смысл для оптимизации производительности (например, flights, boarding_passes).
- Обоснуйте свой выбор: почему именно эта таблица требует секционирования? Какой тип данных является ключевым для секционирования?

Таблица bookings безусловно большая, но таблица boarding_passes в разы больше и наполнена достаточно важнными данными, которая может использоваться, например, при печати билетов и сопоставлении пассажира и его места в самолете на конкретном борту.

```
demo=# select count(book_ref) from bookings.bookings;
  count
---------
 2111110
(1 row)
```
```
demo=# select count(ticket_no) from bookings.boarding_passes;
  count
---------
 7925812
(1 row)
```

Самым очевидным для секционирования полем в таблице boarding_passes является ticket_no, т.к. мы можем создать секции завязанные на числовые диапазоны.

#### Определение типа секционирования:

Определитесь с типом секционирования, которое наилучшим образом подходит для ваших данных:
- По диапазону (например, по дате бронирования или дате рейса).
- По списку (например, по пунктам отправления или по номерам рейсов).
- По хэшированию (для равномерного распределения данных).

В данном случае нам подходит секционирование по диапазону для поля ticket_no. Очевидным недостатком такого метода будет являтся то, что по мере наполнения таблицыв нужно будет создавать новые секции, т.к. номер билета будет постоянно увеличиваться.

```
explain analyze select * from bookings.boarding_passes where ticket_no = '0005433227218';

QUERY PLAN                                                                                                                            |
--------------------------------------------------------------------------------------------------------------------------------------+
Index Scan using boarding_passes_pkey on boarding_passes  (cost=0.56..16.61 rows=3 width=25) (actual time=0.460..0.468 rows=2 loops=1)|
  Index Cond: (ticket_no = '0005433227218'::bpchar)                                                                                   |
Planning Time: 0.057 ms                                                                                                               |
Execution Time: 0.480 ms                                                                                                              |
```

#### Создание секционированной таблицы

- Преобразуйте таблицу в секционированную с выбранным типом секционирования.
- Например, если вы выбрали секционирование по диапазону дат бронирования, создайте секции по месяцам или годам.

1. Для начала проверим наличие секционированных таблиц:

```
demo=# select count(1) from pg_catalog.pg_partitioned_table;
 count
-------
     0
(1 row)
```

2. Проверим, что включен CONSTRAINT_EXCLUSION, иначе запросы не будут оптимизироваться.

```
demo=# show CONSTRAINT_EXCLUSION;
 constraint_exclusion
----------------------
 partition
(1 row)
```

3. Посмотрим на диапазон значений

Начало:

```
demo=# SELECT ticket_no, flight_id, boarding_no, seat_no
FROM bookings.boarding_passes
order by ticket_no asc
limit 3;
   ticket_no   | flight_id | boarding_no | seat_no
---------------+-----------+-------------+---------
 0005432000284 |    187662 |          16 | 10C
 0005432000285 |    187570 |           4 | 2A
 0005432000286 |    187570 |          19 | 10E
(3 rows)
```

Конец:

```
demo=# SELECT ticket_no, flight_id, boarding_no, seat_no
FROM bookings.boarding_passes
order by ticket_no desc
limit 3;
   ticket_no   | flight_id | boarding_no | seat_no
---------------+-----------+-------------+---------
 0005435999484 |     40177 |          33 | 21E
 0005435999483 |     40177 |          60 | 7F
 0005435999482 |     40177 |           3 | 8A
(3 rows)
```

Общее количество уникальных значений:

```
demo=# SELECT COUNT(DISTINCT ticket_no) FROM bookings.boarding_passes;
  count
---------
 2821958
(1 row)
```

Описание таблицы:

```
demo=# \d bookings.boarding_passes
                  Table "bookings.boarding_passes"
   Column    |         Type         | Collation | Nullable | Default
-------------+----------------------+-----------+----------+---------
 ticket_no   | character(13)        |           | not null |
 flight_id   | integer              |           | not null |
 boarding_no | integer              |           | not null |
 seat_no     | character varying(4) |           | not null |
```

Попробуем разбить на секции по 500к

4. Создадим секционированную таблицу:

```
demo=# CREATE TABLE bookings.boarding_passes_p (
  ticket_no  character(13),
   flight_id  integer,
   boarding_no  integer,
   seat_no  character varying(4)
   ) PARTITION BY RANGE(ticket_no);
CREATE TABLE
```

5. Создадим 8 партиций 

```
demo=# CREATE TABLE bookings.boarding_passes_p_0 PARTITION OF bookings.boarding_passes_p FOR VALUES FROM ('0005432000000') TO ('0005432500000');
CREATE TABLE bookings.boarding_passes_p_1 PARTITION OF bookings.boarding_passes_p FOR VALUES FROM ('0005432500000') TO ('0005433000000');
CREATE TABLE bookings.boarding_passes_p_2 PARTITION OF bookings.boarding_passes_p FOR VALUES FROM ('0005433000000') TO ('0005433500000');
CREATE TABLE bookings.boarding_passes_p_3 PARTITION OF bookings.boarding_passes_p FOR VALUES FROM ('0005433500000') TO ('0005434000000');
CREATE TABLE bookings.boarding_passes_p_4 PARTITION OF bookings.boarding_passes_p FOR VALUES FROM ('0005434000000') TO ('0005434500000');
CREATE TABLE bookings.boarding_passes_p_5 PARTITION OF bookings.boarding_passes_p FOR VALUES FROM ('0005434500000') TO ('0005435000000');
CREATE TABLE bookings.boarding_passes_p_6 PARTITION OF bookings.boarding_passes_p FOR VALUES FROM ('0005435000000') TO ('0005435500000');
CREATE TABLE bookings.boarding_passes_p_7 PARTITION OF bookings.boarding_passes_p FOR VALUES FROM ('0005435500000') TO ('0005436000000');
CREATE TABLE
CREATE TABLE
CREATE TABLE
CREATE TABLE
CREATE TABLE
CREATE TABLE
CREATE TABLE
CREATE TABLE
```

Итого получим таблицу:

```
demo=# \d+ bookings.boarding_passes_p;

                                      Partitioned table "bookings.boarding_passes_p"
   Column    |         Type         | Collation | Nullable | Default | Storage  | Compression | Stats target | Description
-------------+----------------------+-----------+----------+---------+----------+-------------+--------------+-------------
 ticket_no   | character(13)        |           |          |         | extended |             |              |
 flight_id   | integer              |           |          |         | plain    |             |              |
 boarding_no | integer              |           |          |         | plain    |             |              |
 seat_no     | character varying(4) |           |          |         | extended |             |              |
Partition key: RANGE (ticket_no)
Partitions: bookings.boarding_passes_p_0 FOR VALUES FROM ('0005432000000') TO ('0005432500000'),
            bookings.boarding_passes_p_1 FOR VALUES FROM ('0005432500000') TO ('0005433000000'),
            bookings.boarding_passes_p_2 FOR VALUES FROM ('0005433000000') TO ('0005433500000'),
            bookings.boarding_passes_p_3 FOR VALUES FROM ('0005433500000') TO ('0005434000000'),
            bookings.boarding_passes_p_4 FOR VALUES FROM ('0005434000000') TO ('0005434500000'),
            bookings.boarding_passes_p_5 FOR VALUES FROM ('0005434500000') TO ('0005435000000'),
            bookings.boarding_passes_p_6 FOR VALUES FROM ('0005435000000') TO ('0005435500000'),
            bookings.boarding_passes_p_7 FOR VALUES FROM ('0005435500000') TO ('0005436000000')

```
#### Миграция данных:

- Перенесите существующие данные из исходной таблицы в секционированную структуру.
- Убедитесь, что все данные правильно распределены по секциям.

1. Перенесем данные:
   
```
demo=# INSERT INTO bookings.boarding_passes_p SELECT * FROM bookings.boarding_passes;
INSERT 0 7925812
```

2. Проверим результат:

```
demo=#  SELECT
    table_name,
    pg_size_pretty(table_size) AS table_size,
    pg_size_pretty(indexes_size) AS indexes_size,
    pg_size_pretty(total_size) AS total_size
FROM (
    SELECT
        table_name,
        pg_table_size(table_name) AS table_size,
        pg_indexes_size(table_name) AS indexes_size,
        pg_total_relation_size(table_name) AS total_size
    FROM (
        SELECT ('"' || table_schema || '"."' || table_name || '"') AS table_name
        FROM information_schema.tables
        WHERE table_name like 'boarding_passes_%'
    ) AS all_tables
    ORDER BY total_size DESC
) AS pretty_sizes;
            table_name            | table_size | indexes_size | total_size
----------------------------------+------------+--------------+------------
 "bookings"."boarding_passes_p_4" | 131 MB     | 0 bytes      | 131 MB
 "bookings"."boarding_passes_p_3" | 122 MB     | 0 bytes      | 122 MB
 "bookings"."boarding_passes_p_2" | 121 MB     | 0 bytes      | 121 MB
 "bookings"."boarding_passes_p_1" | 119 MB     | 0 bytes      | 119 MB
 "bookings"."boarding_passes_p_5" | 115 MB     | 0 bytes      | 115 MB
 "bookings"."boarding_passes_p_7" | 113 MB     | 0 bytes      | 113 MB
 "bookings"."boarding_passes_p_6" | 110 MB     | 0 bytes      | 110 MB
 "bookings"."boarding_passes_p_0" | 81 MB      | 0 bytes      | 81 MB
 "bookings"."boarding_passes_p"   | 0 bytes    | 0 bytes      | 0 bytes
(9 rows)
```





Оптимизация запросов:
Проверьте, как секционирование влияет на производительность запросов. Выполните несколько выборок данных до и после секционирования для оценки времени выполнения.
Оптимизируйте запросы при необходимости (например, добавьте индексы на ключевые столбцы).
Тестирование решения:
Протестируйте секционирование, выполняя несколько запросов к секционированной таблице.
Проверьте, что операции вставки, обновления и удаления работают корректно.
Документирование:
Добавьте комментарии к коду, поясняющие выбранный тип секционирования и шаги его реализации.
Опишите, как секционирование улучшает производительность запросов и как оно может быть полезно в реальных условиях.
Критерии оценивания:
Корректность секционирования – таблица должна быть разделена логично и эффективно.
Выбор типа секционирования – обоснование выбранного типа (например, секционирование по диапазону дат рейсов или по месту отправления/прибытия).
Работоспособность решения – код должен успешно выполнять секционирование без ошибок.
Оптимизация запросов – после секционирования, запросы к таблице должны быть оптимизированы (например, быстрее выполняться для конкретных диапазонов).
Комментирование – код должен содержать поясняющие комментарии, объясняющие выбор секционирования и основные шаги.
