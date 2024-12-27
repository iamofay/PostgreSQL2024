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

Заполним витрину старым способом:

```
postgres=# insert into good_sum_mart SELECT G.good_name, sum(G.good_price * S.sales_qty)
FROM goods G
INNER JOIN sales S ON S.good_id = G.goods_id
GROUP BY G.good_name;
INSERT 0 2
```

Результат:

```
postgres=# select * from good_sum_mart ;
        good_name         |   sum_sale
--------------------------+--------------
 Автомобиль Ferrari FXX K | 185000000.01
 Спички хозайственные     |        65.50
(2 rows)
```

Создадим функцию, которая будем изменять витрину

```
postgres=# CREATE OR REPLACE FUNCTION pract_functions.v_fill()
 RETURNS trigger
 LANGUAGE plpgsql
AS $xxx$
declare
  v_csm good_sum_mart.sum_sale%type;
  v_nsm     good_sum_mart.sum_sale%type;
  v_gp        goods.good_price%type;
  v_gn            goods.good_name%type;
  v_record                record;
begin
  -- Получим текущую статистику продаж товара цены и наименование
  select coalesce(v.sum_sale, 0), g.good_price, g.good_name
  into v_csm, v_gp, v_gn
  from goods g
  left join good_sum_mart v on v.good_name = g.good_name
  where g.goods_id = coalesce(new.good_id, old.good_id);

  case TG_OP
    -- Для вставки добавляем к текущему значению
    when 'INSERT' then
      v_nsm := v_csm + new.sales_qty * v_gp;
      v_record      := new;
    -- Для обновления вычитаем старое значение из текущего и добавляем новое
    when 'UPDATE' then
      v_nsm := v_csm - old.sales_qty * v_gp + new.sales_qty * v_gp;
      v_record      := new;
    -- Для удаления вычитаем старое значение из текущего значения
    when 'DELETE' then
      v_nsm := v_csm - old.sales_qty * v_gp;
      v_record      := old;
  end case;

RAISE NOTICE 'Good name: %', v_gn;
RAISE NOTICE 'New total sale %', v_nsm;

MERGE INTO good_sum_mart AS v
USING (SELECT v_gn as gn, v_nsm as nsm ) AS t ON v.good_name = t.gn
WHEN NOT MATCHED THEN
   INSERT (good_name, sum_sale)
   VALUES(t.gn, t.nsm)
WHEN MATCHED  THEN
   UPDATE SET sum_sale = t.nsm;

return v_record;
end;
$xxx$;
CREATE FUNCTION
```

Создадим вызов триггерной функции:

```
postgres=# CREATE or REPLACE TRIGGER fill_view
AFTER
INSERT OR UPDATE OR DELETE
ON pract_functions.sales
FOR EACH ROW
EXECUTE FUNCTION pract_functions.v_fill();
CREATE TRIGGER
```

Проверим результат в витрине:

```
postgres=#  select v.*
  from goods g
  join good_sum_mart v on g.good_name = v.good_name
 where g.goods_id = 1;
      good_name       | sum_sale
----------------------+----------
 Спички хозайственные |    65.50
(1 row)
```

Добавим продажу спичек:

```
postgres=# INSERT INTO sales (good_id, sales_qty) VALUES (1, 70);
NOTICE:  Good name: Спички хозайственные
NOTICE:  New total sale 100.50
INSERT 0 1
```

Посмотрим результат:

```
postgres=# select v.*
  from goods g
  join good_sum_mart v on g.good_name = v.good_name
 where g.goods_id = 1;
      good_name       | sum_sale
----------------------+----------
 Спички хозайственные |   100.50
(1 row)
```

Обновим данные о продажах

```
postgres=#  update sales set sales_qty = 69 where sales_id = 5;
NOTICE:  Good name: Спички хозайственные
NOTICE:  New total sale 100.00
UPDATE 1
```

```
postgres=# select v.*
  from goods g
  join good_sum_mart v on g.good_name = v.good_name
 where g.goods_id = 1;
      good_name       | sum_sale
----------------------+----------
 Спички хозайственные |   100.00
(1 row)
```

Удалим продажу

```
postgres=# delete from sales where sales_id = 2;
NOTICE:  Good name: Спички хозайственные
NOTICE:  New total sale 99.50
DELETE 1
```

```
postgres=# select v.*
  from goods g
  join good_sum_mart v on g.good_name = v.good_name
 where g.goods_id = 1;
      good_name       | sum_sale
----------------------+----------
 Спички хозайственные |    99.50
(1 row)
```

#### Чем такая схема (витрина+триггер) предпочтительнее отчета, создаваемого "по требованию" (кроме производительности)?

Посмотрим, что будет, если произойдет изменение ценыю.

Текущее состояние:

```
postgres=# select * from goods ;
 goods_id |        good_name         |  good_price
----------+--------------------------+--------------
        1 | Спички хозайственные     |         0.50
        2 | Автомобиль Ferrari FXX K | 185000000.01
(2 rows)
```

```
postgres=# select * from sales ;
 sales_id | good_id |          sales_time           | sales_qty
----------+---------+-------------------------------+-----------
        1 |       1 | 2024-12-27 10:39:01.533552+03 |        10
        3 |       1 | 2024-12-27 10:39:01.533552+03 |       120
        4 |       2 | 2024-12-27 10:39:01.533552+03 |         1
        5 |       1 | 2024-12-27 10:43:05.226206+03 |        69
(4 rows)
```

```
postgres=#  select * from good_sum_mart ;
        good_name         |   sum_sale
--------------------------+--------------
 Автомобиль Ferrari FXX K | 185000000.01
 Спички хозайственные     |        99.50
(2 rows)
```

Изменим цену на авто:

```
postgres=# update goods set good_price = 1600000.50 where goods_id = 2;
UPDATE 1
```

```
postgres=# select * from goods ;
 goods_id |        good_name         | good_price
----------+--------------------------+------------
        1 | Спички хозайственные     |       0.50
        2 | Автомобиль Ferrari FXX K | 1600000.50
(2 rows)
```

Добавим прожажи

```
postgres=# INSERT INTO sales (good_id, sales_qty) VALUES (2, 2);
NOTICE:  Good name: Автомобиль Ferrari FXX K
NOTICE:  New total sale 188200001.01
INSERT 0 1
```

Сейчас все хорошо

```
postgres=# select v.*
  from goods g
  join good_sum_mart v on g.good_name = v.good_name
 where g.goods_id = 2;
        good_name         |   sum_sale
--------------------------+--------------
 Автомобиль Ferrari FXX K | 188200001.01
(1 row)
```

Но если мы обновим информацию, наш отчет уже не будет содержать корректной информации

```
postgres=# truncate table good_sum_mart;
TRUNCATE TABLE
postgres=# insert into good_sum_mart SELECT G.good_name, sum(G.good_price * S.sales_qty)
postgres-# FROM goods G
INNER JOIN sales S ON S.good_id = G.goods_id
GROUP BY G.good_name;
INSERT 0 2
postgres=# select v.*
  from goods g
  join good_sum_mart v on g.good_name = v.good_name
 where g.goods_id = 2;
        good_name         |  sum_sale
--------------------------+------------
 Автомобиль Ferrari FXX K | 4800001.50
(1 row)
```

Чтобы такого не было необходимо хранить всю историю цен, что и помогает решить схема (витрина+триггер)
