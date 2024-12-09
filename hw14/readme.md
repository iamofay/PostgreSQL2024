### Триггеры, поддержка заполнения витрин

Цель:
- Создать триггер для поддержки витрины в актуальном состоянии.

#### Исходное ДЗ https://disk.yandex.ru/d/l70AvknAepIJXQ:

#### Результаты выполнения команд

```
postgres=# DROP SCHEMA IF EXISTS pract_functions CASCADE;
CREATE SCHEMA pract_functions;
NOTICE:  schema "pract_functions" does not exist, skipping
DROP SCHEMA
CREATE SCHEMA
postgres=# SET search_path = pract_functions, publ
postgres-# ;
SET
postgres=# CREATE TABLE goods
(
    goods_id    integer PRIMARY KEY,
    good_name   varchar(63) NOT NULL,
    good_price  numeric(12, 2) NOT NULL CHECK (good_price > 0.0)
);
CREATE TABLE
postgres=# INSERT INTO goods (goods_id, good_name, good_price)
VALUES  (1, 'Спички хозайственные', .50),
                (2, 'Автомобиль Ferrari FXX K', 185000000.01);
INSERT 0 2
postgres=# CREATE TABLE sales
(
    sales_id    integer GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    good_id     integer REFERENCES goods (goods_id),
    sales_time  timestamp with time zone DEFAULT now(),
    sales_qty   integer CHECK (sales_qty > 0)
);
CREATE TABLE
postgres=# INSERT INTO sales (good_id, sales_qty) VALUES (1, 10), (1, 1), (1, 120), (2, 1);
INSERT 0 4
postgres=# SELECT G.good_name, sum(G.good_price * S.sales_qty)
FROM goods G
INNER JOIN sales S ON S.good_id = G.goods_id
GROUP BY G.good_name;
        good_name         |     sum
--------------------------+--------------
 Автомобиль Ferrari FXX K | 185000000.01
 Спички хозайственные     |        65.50
(2 rows)

postgres=# CREATE TABLE good_sum_mart
(
        good_name   varchar(63) NOT NULL,
        sum_sale        numeric(16, 2)NOT NULL
);
CREATE TABLE
```

#### Создать триггер на таблице продаж, для поддержки данных в витрине в актуальном состоянии (вычисляющий при каждой продаже сумму и записывающий её в витрину)


Подсказка: не забыть, что кроме INSERT есть еще UPDATE и DELETE

Задание со звездочкой*

Чем такая схема (витрина+триггер) предпочтительнее отчета, создаваемого "по требованию" (кроме производительности)?
Подсказка: В реальной жизни возможны изменения цен.
