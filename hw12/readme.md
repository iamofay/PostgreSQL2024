### Работа с join'ами, статистикой

Цель:
- знать и уметь применять различные виды join'ов
- строить и анализировать план выполенения запроса
- оптимизировать запрос
- уметь собирать и анализировать статистику для таблицы

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

#### Реализовать прямое соединение двух или более таблиц

Посмотрим, какое количество вылетов было у каждого самолета

```
select a.aircraft_code, a.model, count(f.*) количество_рейсов
from bookings.aircrafts_data a
         join bookings.flights f on a.aircraft_code = f.aircraft_code
group by 1
order by 3 desc

aircraft_code|model                                                     |количество_рейсов|
-------------+----------------------------------------------------------+-----------------+
CN1          |{"en": "Cessna 208 Caravan", "ru": "Сессна 208 Караван"}  |             9273|
CR2          |{"en": "Bombardier CRJ-200", "ru": "Бомбардье CRJ-200"}   |             9048|
SU9          |{"en": "Sukhoi Superjet-100", "ru": "Сухой Суперджет-100"}|             8504|
321          |{"en": "Airbus A321-200", "ru": "Аэробус A321-200"}       |             1952|
733          |{"en": "Boeing 737-300", "ru": "Боинг 737-300"}           |             1274|
319          |{"en": "Airbus A319-100", "ru": "Аэробус A319-100"}       |             1239|
763          |{"en": "Boeing 767-300", "ru": "Боинг 767-300"}           |             1221|
773          |{"en": "Boeing 777-300", "ru": "Боинг 777-300"}           |              610|
```

#### Реализовать левостороннее (или правостороннее) соединение двух или более таблиц

Попробуем добавить к нашему запросу left join и получим результат даже лучше, т.к. получим результаты по рейсу, который не имел вылетов.

```
select a.aircraft_code, a.model, count(f.*) количество_рейсов
from bookings.aircrafts_data a
        left join bookings.flights f on a.aircraft_code = f.aircraft_code
group by 1
order by 3 desc

aircraft_code|model                                                     |количество_рейсов|
-------------+----------------------------------------------------------+-----------------+
CN1          |{"en": "Cessna 208 Caravan", "ru": "Сессна 208 Караван"}  |             9273|
CR2          |{"en": "Bombardier CRJ-200", "ru": "Бомбардье CRJ-200"}   |             9048|
SU9          |{"en": "Sukhoi Superjet-100", "ru": "Сухой Суперджет-100"}|             8504|
321          |{"en": "Airbus A321-200", "ru": "Аэробус A321-200"}       |             1952|
733          |{"en": "Boeing 737-300", "ru": "Боинг 737-300"}           |             1274|
319          |{"en": "Airbus A319-100", "ru": "Аэробус A319-100"}       |             1239|
763          |{"en": "Boeing 767-300", "ru": "Боинг 767-300"}           |             1221|
773          |{"en": "Boeing 777-300", "ru": "Боинг 777-300"}           |              610|
320          |{"en": "Airbus A320-200", "ru": "Аэробус A320-200"}       |                0|
```

Попробуем еще что нибудь, например посчитаем количество человек, которые были перевезены этими самолетами

```
select a.aircraft_code, a.model, count(tf.*) Количество_пассажиров
from bookings.aircrafts_data a
         left join bookings.flights f on a.aircraft_code = f.aircraft_code
         left join bookings.ticket_flights tf on f.flight_id = tf.flight_id 
group by 1
order by 3 desc

aircraft_code|model                                                     |Количество_пассажиров|
-------------+----------------------------------------------------------+---------------------+
SU9          |{"en": "Sukhoi Superjet-100", "ru": "Сухой Суперджет-100"}|               365698|
CR2          |{"en": "Bombardier CRJ-200", "ru": "Бомбардье CRJ-200"}   |               150122|
773          |{"en": "Boeing 777-300", "ru": "Боинг 777-300"}           |               144376|
763          |{"en": "Boeing 767-300", "ru": "Боинг 767-300"}           |               124774|
321          |{"en": "Airbus A321-200", "ru": "Аэробус A321-200"}       |               107129|
733          |{"en": "Boeing 737-300", "ru": "Боинг 737-300"}           |                86102|
319          |{"en": "Airbus A319-100", "ru": "Аэробус A319-100"}       |                52853|
CN1          |{"en": "Cessna 208 Caravan", "ru": "Сессна 208 Караван"}  |                14672|
320          |{"en": "Airbus A320-200", "ru": "Аэробус A320-200"}       |                    0|
```

#### Реализовать кросс соединение двух или более таблиц

Перечислим всех пассажиров конкретного рейса

```
select a.aircraft_code, t.passenger_name
from bookings.aircrafts_data a 
	cross join  bookings.tickets t
where a.aircraft_code ='SU9'
limit 10;

aircraft_code|passenger_name     |
-------------+-------------------+
SU9          |VALERIY TIKHONOV   |
SU9          |EVGENIYA ALEKSEEVA |
SU9          |ARTUR GERASIMOV    |
SU9          |ALINA VOLKOVA      |
SU9          |MAKSIM ZHUKOV      |
SU9          |NIKOLAY EGOROV     |
SU9          |TATYANA KUZNECOVA  |
SU9          |IRINA ANTONOVA     |
SU9          |VALENTINA KUZNECOVA|
SU9          |POLINA ZHURAVLEVA  |
```

#### Реализовать полное соединение двух или более таблиц

Можем посмотреть какие самолеты вылетали из каких аэропортов, и тут мы увидим, что 320 рейс никуда и не летал

```
select DISTINCT ad2.aircraft_code, f.departure_airport 
from bookings.flights f
	full join bookings.aircrafts_data ad2 on f.aircraft_code = ad2.aircraft_code
order by aircraft_code
limit 30;

aircraft_code|departure_airport|
-------------+-----------------+
319          |ABA              |
319          |DME              |
319          |ARH              |
319          |CNN              |
319          |PEE              |
319          |HTA              |
319          |KEJ              |
319          |LED              |
319          |DYR              |
319          |VKO              |
319          |TOF              |
319          |UFA              |
319          |KZN              |
319          |YKS              |
319          |UUS              |
319          |AER              |
319          |IKT              |
319          |KJA              |
319          |UUD              |
319          |OVB              |
319          |KHV              |
319          |SVO              |
319          |BTK              |
319          |ASF              |
319          |ROV              |
320          |                 |
321          |IKT              |
321          |SVO              |
321          |LED              |
321          |VKO              |
```

####Реализовать запрос, в котором будут использованы разные типы соединений

Модем посмотреть какими рейсами летал на самолете один из пассажиров

```
select a.aircraft_code, t.passenger_name, tf.flight_id 
from bookings.aircrafts_data a 
	cross join  bookings.tickets t 
	join bookings.ticket_flights tf on t.ticket_no = tf.ticket_no 
where a.aircraft_code ='SU9' and t.passenger_name = 'ANDREY YAKOVLEV'
order by flight_id desc
limit 30;

aircraft_code|passenger_name |flight_id|
-------------+---------------+---------+
SU9          |ANDREY YAKOVLEV|    32657|
SU9          |ANDREY YAKOVLEV|    32576|
SU9          |ANDREY YAKOVLEV|    32479|
SU9          |ANDREY YAKOVLEV|    32467|
SU9          |ANDREY YAKOVLEV|    32457|
SU9          |ANDREY YAKOVLEV|    32455|
SU9          |ANDREY YAKOVLEV|    32376|
SU9          |ANDREY YAKOVLEV|    31878|
SU9          |ANDREY YAKOVLEV|    31722|
SU9          |ANDREY YAKOVLEV|    31331|
SU9          |ANDREY YAKOVLEV|    31248|
SU9          |ANDREY YAKOVLEV|    31109|
SU9          |ANDREY YAKOVLEV|    30702|
SU9          |ANDREY YAKOVLEV|    30676|
SU9          |ANDREY YAKOVLEV|    30625|
SU9          |ANDREY YAKOVLEV|    30609|
SU9          |ANDREY YAKOVLEV|    30535|
SU9          |ANDREY YAKOVLEV|    30530|
SU9          |ANDREY YAKOVLEV|    30477|
SU9          |ANDREY YAKOVLEV|    30361|
SU9          |ANDREY YAKOVLEV|    30336|
SU9          |ANDREY YAKOVLEV|    30157|
SU9          |ANDREY YAKOVLEV|    29827|
SU9          |ANDREY YAKOVLEV|    29753|
SU9          |ANDREY YAKOVLEV|    29532|
SU9          |ANDREY YAKOVLEV|    29521|
SU9          |ANDREY YAKOVLEV|    29216|
SU9          |ANDREY YAKOVLEV|    28961|
SU9          |ANDREY YAKOVLEV|    28895|
SU9          |ANDREY YAKOVLEV|    28746|
```

####Задание со звездочкой* Придумайте 3 своих метрики на основе показанных представлений, отправьте их через ЛК, а так же поделитесь с коллегами в слаке


