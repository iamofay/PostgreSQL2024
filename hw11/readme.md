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

Возьмем таблицу ticket_flights, т.к. она самая большая. В ней уже есть btree индекс ticket_flights_pkey завязанный на ticket_no и flight_id.

```
demo=# \d "ticket_flights"
                     Table "bookings.ticket_flights"
     Column      |         Type          | Collation | Nullable | Default
-----------------+-----------------------+-----------+----------+---------
 ticket_no       | character(13)         |           | not null |
 flight_id       | integer               |           | not null |
 fare_conditions | character varying(10) |           | not null |
 amount          | numeric(10,2)         |           | not null |
Indexes:
    "ticket_flights_pkey" PRIMARY KEY, btree (ticket_no, flight_id)
```

Такой индекс мог бы быть создан запросом, поэтому попробуем для начала использовать его.

```
CREATE UNIQUE INDEX ticket_flights_pkey ON bookings.ticket_flights USING btree (ticket_no, flight_id)
```

#### Прислать текстом результат команды explain, в которой используется данный индекс. Попробуем разные варианты запроса.

```
demo=# explain analyze select * from "ticket_flights" where "ticket_no" = '0005432211370' 

                                                           QUERY PLAN        
--------------------------------------------------------------------------------------------------------------------------------------
 Index Scan using ticket_flights_pkey on ticket_flights  (cost=0.42..16.47 rows=3 width=32) (actual time=0.069..0.069 rows=1 loops=1)
   Index Cond: (ticket_no = '0005432211370'::bpchar)
 Planning Time: 0.250 ms
 Execution Time: 0.080 ms
(4 rows)
```

```
demo=# explain analyze select * from "ticket_flights" where "ticket_no" = '0005432211370' and flight_id = '30625';

                                                            QUERY PLAN         
-------------------------------------------------------------------------------------------------------------------------------------
 Index Scan using ticket_flights_pkey on ticket_flights  (cost=0.42..8.45 rows=1 width=32) (actual time=0.015..0.015 rows=1 loops=1)
   Index Cond: ((ticket_no = '0005432211370'::bpchar) AND (flight_id = 30625))
 Planning Time: 0.301 ms
 Execution Time: 0.050 ms
(4 rows)
```

```
demo=# explain analyze select * from "ticket_flights" where flight_id = '30625';

                                                           QUERY PLAN           
--------------------------------------------------------------------------------------------------------------------------------
 Gather  (cost=1000.00..15168.19 rows=67 width=32) (actual time=0.424..37.374 rows=92 loops=1)
   Workers Planned: 2
   Workers Launched: 2
   ->  Parallel Seq Scan on ticket_flights  (cost=0.00..14161.49 rows=28 width=32) (actual time=18.511..30.020 rows=31 loops=3)
         Filter: (flight_id = 30625)
         Rows Removed by Filter: 348545
 Planning Time: 0.074 ms
 Execution Time: 37.429 ms
(8 rows)
```

И тут мы можем наблюдать интересную картину, что в случае поиска по flight_id индекс не используется, и поиск осуществляется c помощью Parallel Seq Scan и параллельно запускает 2 рабочих процесса, а flight_id = 30625 применяется как фильтр к запросу, в отличии от остальных запросов где для поиска использовался уже индекс.

#### Реализовать индекс для полнотекстового поиска

Для решения этой задачи попробуеи использовать таблицу tickets, которая содержит информацию из пассажирских билетов. Таблица уже имеет индекс tickets_pkey.

```
demo=# \d "tickets"
                        Table "bookings.tickets"
     Column     |         Type          | Collation | Nullable | Default
----------------+-----------------------+-----------+----------+---------
 ticket_no      | character(13)         |           | not null |
 book_ref       | character(6)          |           | not null |
 passenger_id   | character varying(20) |           | not null |
 passenger_name | text                  |           | not null |
 contact_data   | jsonb                 |           |          |
Indexes:
    "tickets_pkey" PRIMARY KEY, btree (ticket_no)
```

1. Создадим свой индекс завязанный на book_ref и passenger_name, но для начала добавим колонку с данными в формате TSVECTOR, который необходим для создания GiN индекса

```
demo=# ALTER TABLE tickets ADD COLUMN content_tsvector TSVECTOR GENERATED ALWAYS AS (to_tsvector('english', book_ref) || to_tsvector('english', passenger_name)) STORED;
ALTER TABLE
demo=# SELECT * FROM tickets;
   ticket_no   | book_ref | passenger_id |     passenger_name      |                                     contact_data                                      |                content_tsvector
---------------+----------+--------------+-------------------------+---------------------------------------------------------------------------------------+-------------------------------------------------
 0005432000987 | 06B046   | 8149 604011  | VALERIY TIKHONOV        | {"phone": "+70127117011"}                                                             | '06b046':1 'tikhonov':3 'valeriy':2
 0005432000988 | 06B046   | 8499 420203  | EVGENIYA ALEKSEEVA      | {"phone": "+70378089255"}                                                             | '06b046':1 'alekseeva':3 'evgeniya':2
 0005432000989 | E170C3   | 1011 752484  | ARTUR GERASIMOV         | {"phone": "+70760429203"}                                                             | 'artur':2 'e170c3':1 'gerasimov':3
 0005432000990 | E170C3   | 4849 400049  | ALINA VOLKOVA           | {"email": "volkova.alina_03101973@postgrespro.ru", "phone": "+70582584031"}           | 'alina':2 'e170c3':1 'volkova':3
 0005432000991 | F313DD   | 6615 976589  | MAKSIM ZHUKOV           | {"email": "m-zhukov061972@postgrespro.ru", "phone": "+70149562185"}                   | 'f313dd':1 'maksim':2 'zhukov':3
 0005432000992 | F313DD   | 2021 652719  | NIKOLAY EGOROV          | {"phone": "+70791452932"}                                                             | 'eg:
```

2. Теперь создадим сам индекс

```
demo=# CREATE INDEX idx_tickets_content_tsvector ON "tickets" USING gin (content_tsvector);
CREATE INDEX

demo=# \d "tickets"
                                                                                       Table "bookings.tickets"
      Column      |         Type          | Collation | Nullable |                                                               Default                        
------------------+-----------------------+-----------+----------+-------------------------------------------------------------------------------------------------------------------------------------
 ticket_no        | character(13)         |           | not null |
 book_ref         | character(6)          |           | not null |
 passenger_id     | character varying(20) |           | not null |
 passenger_name   | text                  |           | not null |
 contact_data     | jsonb                 |           |          |
 content_tsvector | tsvector              |           |          | generated always as (to_tsvector('english'::regconfig, book_ref::text) || to_tsvector('english'::regconfig, passenger_name)) stored
Indexes:
    "tickets_pkey" PRIMARY KEY, btree (ticket_no)
    "idx_tickets_content_tsvector" gin (content_tsvector)
```

3. Посмотрим на EXPLAIN

```
demo=# EXPLAIN SELECT * FROM tickets WHERE content_tsvector @@ to_tsquery('english', 'borisova & 8e6*');

                                         QUERY PLAN                             
--------------------------------------------------------------------------------------------
 Bitmap Heap Scan on tickets  (cost=28.07..63.30 rows=9 width=136)
   Recheck Cond: (content_tsvector @@ '''borisova'' & ''8e6'''::tsquery)
   ->  Bitmap Index Scan on idx_tickets_content_tsvector  (cost=0.00..28.07 rows=9 width=0)
         Index Cond: (content_tsvector @@ '''borisova'' & ''8e6'''::tsquery)
(4 rows)
```

4. Попробуем select

```
demo=# SELECT * FROM tickets WHERE content_tsvector @@ to_tsquery('english', 'borisova & 8e6*');

   ticket_no   | book_ref | passenger_id | passenger_name |       contact_data        |           content_tsvector
---------------+----------+--------------+----------------+---------------------------+---------------------------------------
 0005432001051 | 8E6BB3   | 5582 658715  | ANNA BORISOVA  | {"phone": "+70282151122"} | '8e6':1 'anna':3 'bb3':2 'borisova':4
(1 row)
```

#### Реализовать индекс на часть таблицы или индекс на поле с функцией

Для этого используем таблицу flights, в таблице присутствуют данные о времени вылета самолета scheduled_departure

```
demo=# SELECT * FROM flights;
 flight_id | flight_no |  scheduled_departure   |   scheduled_arrival    | departure_airport | arrival_airport |  status   | aircraft_code |    actual_departure    |     actual_arrival
-----------+-----------+------------------------+------------------------+-------------------+-----------------+-----------+---------------+------------------------+------------------------
      1185 | PG0134    | 2017-09-10 09:50:00+03 | 2017-09-10 14:55:00+03 | DME               | BTK             | Scheduled | 319           |                        |
      3979 | PG0052    | 2017-08-25 14:50:00+03 | 2017-08-25 17:35:00+03 | VKO               | HMA             | Scheduled | CR2           |                        |
      4739 | PG0561    | 2017-09-05 12:30:00+03 | 2017-09-05 14:15:00+03 | VKO               | AER             | Scheduled | 763           |                        |
      5502 | PG0529    | 2017-09-12 09:50:00+03 | 2017-09-12 11:20:00+03 | SVO               | UFA             | Scheduled | 763           |                        |
      6938 | PG0461    | 2017-09-04 12:25:00+03 | 2017-09-04 13:20:00+03 | SVO               | ULV             | Scheduled | SU9           |                        |
      7784 | PG0667    | 2017-09-10 15:00:00+03 | 2017-09-10 17:30:00+03 | SVO               | KRO             | Scheduled | CR2           |                    :
```

1. Добавим булевый столбец в котором будет информация true, если был дневной вылет (BETWEEN '09:00:00' AND '21:00:00')

```
demo=# alter table "flights" ADD COLUMN day bool DEFAULT 'false' ;
ALTER TABLE
```
  
2. Добавим данных в новый столбец

```
demo=# update "flights" SET day = 'true' where CAST(scheduled_departure AS TIME) BETWEEN '09:00:00' AND '21:00:00';
UPDATE 29143
```

3. Добавим индекс к части таблицы

```
demo=# create index ON "flights" (day) where day = 'true';
CREATE INDEX
demo=# \d "flights"
                                              Table "bookings.flights"
       Column        |           Type           | Collation | Nullable |                  Default
---------------------+--------------------------+-----------+----------+--------------------------------------------
 flight_id           | integer                  |           | not null | nextval('flights_flight_id_seq'::regclass)
 flight_no           | character(6)             |           | not null |
 scheduled_departure | timestamp with time zone |           | not null |
 scheduled_arrival   | timestamp with time zone |           | not null |
 departure_airport   | character(3)             |           | not null |
 arrival_airport     | character(3)             |           | not null |
 status              | character varying(20)    |           | not null |
 aircraft_code       | character(3)             |           | not null |
 actual_departure    | timestamp with time zone |           |          |
 actual_arrival      | timestamp with time zone |           |          |
 day                 | boolean                  |           |          | false
Indexes:
    "flights_pkey" PRIMARY KEY, btree (flight_id)
    "flights_day_idx" btree (day) WHERE day = true

```

4. Посмотрим EXPLAIN

```
demo=# explain analyze select count(*) from "flights" where day;

                                                                  QUERY PLAN    
----------------------------------------------------------------------------------------------------------------------------------------------
 Aggregate  (cost=618.68..618.69 rows=1 width=8) (actual time=2.124..2.124 rows=1 loops=1)
   ->  Index Only Scan using flights_day_idx on flights  (cost=0.29..545.74 rows=29174 width=0) (actual time=0.016..1.280 rows=29143 loops=1)
         Heap Fetches: 0
 Planning Time: 0.078 ms
 Execution Time: 2.139 ms
(5 rows)
```

#### Создать индекс на несколько полей

Ранее уже было реализованто в рамках полнотекстового поиска

```
demo=# SELECT * FROM tickets WHERE content_tsvector @@ to_tsquery('english', 'borisova & 8e6*');

   ticket_no   | book_ref | passenger_id | passenger_name |       contact_data        |           content_tsvector
---------------+----------+--------------+----------------+---------------------------+---------------------------------------
 0005432001051 | 8E6BB3   | 5582 658715  | ANNA BORISOVA  | {"phone": "+70282151122"} | '8e6':1 'anna':3 'bb3':2 'borisova':4
(1 row)
```

#### Добавить комментарии к индексам

```
demo=# COMMENT ON INDEX flights_day_idx IS 'Определяет время суток, в которое был совершен вылет (день\ночь)';
COMMENT ON INDEX idx_tickets_content_tsvector IS 'Полнотекстовый индекс для поиска по имени и номеру билета';
COMMENT ON INDEX ticket_flights_pkey IS 'Индекс по прайм ключу таблицы вылетов и номеру вылета';
COMMENT
COMMENT
COMMENT
```

Посмотрим на добавленные комментарии

```
demo=# \di+
                                                                                         List of relations
  Schema  |                   Name                    | Type  |  Owner   |      Table      | Persistence | Access method |  Size   |                           Description
----------+-------------------------------------------+-------+----------+-----------------+-------------+---------------+---------+------------------------------------------------------------------
 bookings | aircrafts_pkey                            | index | postgres | aircrafts_data  | permanent   | btree         | 16 kB   |
 bookings | airports_data_pkey                        | index | postgres | airports_data   | permanent   | btree         | 16 kB   |
 bookings | boarding_passes_flight_id_boarding_no_key | index | postgres | boarding_passes | permanent   | btree         | 12 MB   |
 bookings | boarding_passes_flight_id_seat_no_key     | index | postgres | boarding_passes | permanent   | btree         | 12 MB   |
 bookings | boarding_passes_pkey                      | index | postgres | boarding_passes | permanent   | btree         | 22 MB   |
 bookings | bookings_pkey                             | index | postgres | bookings        | permanent   | btree         | 5784 kB |
 bookings | flights_day_idx                           | index | postgres | flights         | permanent   | btree         | 216 kB  | Определяет время суток, в которое был совершен вылет (день\ночь)
 bookings | flights_flight_no_scheduled_departure_key | index | postgres | flights         | permanent   | btree         | 2056 kB |
 bookings | flights_pkey                              | index | postgres | flights         | permanent   | btree         | 1464 kB |
 bookings | idx_tickets_content_tsvector              | index | postgres | tickets         | permanent   | gin           | 18 MB   | Полнотекстовый индекс для поиска по имени и номеру билета
 bookings | seats_pkey                                | index | postgres | seats           | permanent   | btree         | 48 kB   |
 bookings | ticket_flights_pkey                       | index | postgres | ticket_flights  | permanent   | btree         | 41 MB   | Индекс по прайм ключу таблицы вылетов и номеру вылета
 bookings | tickets_pkey                              | index | postgres | tickets         | permanent   | btree         | 11 MB   |
(13 rows)
```
