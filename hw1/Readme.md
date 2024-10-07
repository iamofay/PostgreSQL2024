### Работа с уровнями изоляции транзакции в PostgreSQL

Цель:
- научиться работать в Яндекс Облаке
- научиться управлять уровнем изолции транзации в PostgreSQL и понимать особенность работы уровней read commited и repeatable read

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

```
![image](https://github.com/user-attachments/assets/afa38444-4ffb-4a91-8f2d-748630c4f31d)
```

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

#### Установка postgresql

```
sudo apt install postgresql
```

#### Запустим psql из под УЗ postgres и откроем 2 удаленных сессии

```
sudo -u postgres psql
```

![image](https://github.com/user-attachments/assets/8914026f-dcb2-420c-ad11-8df3021d8c9e)

#### Выключим autoconnit

```
\set AUTOCOMMIT off
```

![image](https://github.com/user-attachments/assets/4ba16767-4f88-4654-a348-3263b0af18a7)

#### Создадим новую БД hw1 и присоединимся к ней

```
CREATE DATABASE hw1;
\с hw1
```
```
hw1=# \l
                                                       List of databases
   Name    |  Owner   | Encoding | Locale Provider |   Collate   |    Ctype    | ICU Locale |
-----------+----------+----------+-----------------+-------------+-------------+------------+
 hw1       | postgres | UTF8     | libc            | ru_RU.UTF-8 | ru_RU.UTF-8 |            |
 postgres  | postgres | UTF8     | libc            | ru_RU.UTF-8 | ru_RU.UTF-8 |            |
 template0 | postgres | UTF8     | libc            | ru_RU.UTF-8 | ru_RU.UTF-8 |            |
           |          |          |                 |             |             |            |
 template1 | postgres | UTF8     | libc            | ru_RU.UTF-8 | ru_RU.UTF-8 |            |
           |          |          |                 |             |             |            |
(4 rows)
```
![image](https://github.com/user-attachments/assets/1f2564de-4185-4073-90af-3010de26c92b)

#### Создадим таблицу persons и заполним ее данными

```
hw1=# CREATE TABLE persons(id serial, first_name text, second_name text);
```
```
hw1=# \d
               List of relations
 Schema |      Name      |   Type   |  Owner
--------+----------------+----------+----------
 public | persons        | table    | postgres
 public | persons_id_seq | sequence | postgres
(2 rows)
```

#### Заполним таблицу persons данными и закоммитим

```
hw1=# INSERT INTO persons(first_name, second_name) VALUES('ivan','ivanov');
INSERT 0 1
hw1=# INSERT INTO persons(first_name, second_name) VALUES('petr','petrov');
INSERT 0 1
```
```
hw1=*# commit;
COMMIT
```
```
hw1=# SELECT * FROM persons;
 id | first_name | second_name
----+------------+-------------
  1 | ivan       | ivanov
  2 | petr       | petrov
(2 rows)
```

#### Посмотрим текущий уровень изоляции

```
hw1=# show transaction isolation level;
 transaction_isolation
-----------------------
 read committed
(1 row)
```

#### Начнем новые сессии с текущим уровнем изоляции. 

В перой сессии добавим данных в таблицу persons

```
hw1=*# INSERT INTO persons(first_name, second_name) VALUES ('sergey','sergeev');
INSERT 0 1
```

Во второй сессии посмотрим, что можно увидеть в таблице persons

```
hw1=*# select * from persons;
 id | first_name | second_name
----+------------+-------------
  1 | ivan       | ivanov
  2 | petr       | petrov
(2 rows)
```

Во второй сессии мы не видим изменений в данных. Связано это с тем, что READ COMMITTED является уровнем изоляции по умолчанию. Он предотвращает операции чтения "грязных" данных, указывая, что инструкции не могут считывать значения данных, которые были изменены, но еще не зафиксированы другими транзакциями.

#### Завершим транзакцию в первой сессии и уже во второй сможем увидеть данные. Завершим транзакцию во второй сессии.

```
hw1=*# commit;
COMMIT
```
```
hw1=*# select * from persons;
 id | first_name | second_name
----+------------+-------------
  1 | ivan       | ivanov
  2 | petr       | petrov
(2 rows)

hw1=*# select * from persons;
 id | first_name | second_name
----+------------+-------------
  1 | ivan       | ivanov
  2 | petr       | petrov
  3 | sergey     | sergeev
(3 rows)

hw1=*# commit;
COMMIT
```

#### Начнем новые транзакции в обеих сессиях но уже с правами repeatable read

```
hw1=# set transaction isolation level repeatable read;
SET
hw1=*# show transaction isolation level;
 transaction_isolation
-----------------------
 repeatable read
(1 row)
```

#### Добавим данные в таблицу pearsons в первой сессии

```
hw1=*# insert into persons(first_name, second_name) values('sveta', 'svetova');
INSERT 0 1
```

#### Посмотрим изменения через вторую сессию

```
hw1=# set transaction isolation level repeatable read;
SET
hw1=*# select * from persons;
 id | first_name | second_name
----+------------+-------------
  1 | ivan       | ivanov
  2 | petr       | petrov
  3 | sergey     | sergeev
(3 rows)
```

Изменений в таблице не видно, т.к. REPEATABLE READ — это более строгий уровень изоляции, чем READ COMMITTED, при этом включает READ COMMITTED.

#### Завершим транзакцию в первой сессии

```
hw1=*# insert into persons(first_name, second_name) values('sveta', 'svetova');
INSERT 0 1
hw1=*# commit;
COMMIT
```

#### Посмотрим изменения в таблице через вторую сессию

```
hw1=*# select * from persons;
 id | first_name | second_name
----+------------+-------------
  1 | ivan       | ivanov
  2 | petr       | petrov
  3 | sergey     | sergeev
(3 rows)
```

Изменений так же не видно, т.к. REPEATABLE READ включает в себя READ COMMITTED и дополнительно указывает, что никакие другие транзакции не могут изменять или удалять данные, которые были считаны текущей транзакцией, пока текущая транзакция не будет зафиксирована.

#### Попробуем закоммитить транзакцию во второй сессии и теперь уже увидим данные

```
hw1=*# select * from persons;
 id | first_name | second_name
----+------------+-------------
  1 | ivan       | ivanov
  2 | petr       | petrov
  3 | sergey     | sergeev
(3 rows)

hw1=*# commit;
COMMIT
hw1=# select * from persons;
 id | first_name | second_name
----+------------+-------------
  1 | ivan       | ivanov
  2 | petr       | petrov
  3 | sergey     | sergeev
  4 | sveta      | svetova
(4 rows)
```
