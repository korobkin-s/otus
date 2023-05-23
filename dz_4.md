## Работа с базами данных, пользователями и правами

Цель:

-   создание новой базы данных, схемы и таблицы
-   создание роли для чтения данных из созданной схемы созданной базы данных
-   создание роли для чтения и записи из созданной схемы созданной базы данных

  

### Описание/Пошаговая инструкция выполнения домашнего задания:

###### ✅ 1 Создайте новый кластер PostgresSQL 14

```
root@otus:~# pg_config --version
PostgreSQL 14.8 (Ubuntu 14.8-1.pgdg22.04+1)

root@otus:~#
```

###### ✅ 2 Зайдите в созданный кластер под пользователем postgres 

```
root@otus:~# sudo -u postgres psql
could not change directory to "/root": Permission denied
psql (14.8 (Ubuntu 14.8-1.pgdg22.04+1))
Type "help" for help.

postgres=#
```

###### ✅ 3 Создайте новую базу данных testdb

```
postgres=# CREATE DATABASE testdb;
CREATE DATABASE

postgres=#
```

###### ✅ 4 Зайдите в созданную базу данных под пользователем postgres 

```
postgres=# \c testdb
You are now connected to database "testdb" as user "postgres".

testdb=#
```

###### ✅ 5 Создайте новую схему testnm

```
testdb=# CREATE SCHEMA testnm;
CREATE SCHEMA

testdb=#
```

###### ✅ 6 Создайте новую таблицу t1 с одной колонкой c1 типа integer

```
testdb=# CREATE TABLE testnm.t1(c1 integer);
CREATE TABLE

testdb=#
```

###### ✅ 7 Вставьте строку со значением c1=1

```
testdb=# INSERT INTO testnm.t1 values(1);
INSERT 0 1

testdb=#
```

###### ✅ 8 Создайте новую роль readonly

```
testdb=# CREATE role readonly;
CREATE ROLE

testdb=#
```

###### ✅ 9 Дайте новой роли право на подключение к базе данных testdb

```
testdb=# grant connect on DATABASE testdb TO readonly;
GRANT

testdb=#
```

###### ✅ 10 Дайте новой роли право на использование схемы testnm

```
testdb=# grant usage on SCHEMA testnm to readonly;
GRANT

testdb=#
```

###### ✅ 11 Дайте новой роли право на select для всех таблиц схемы testnm

```
testdb=# grant SELECT on all TABLEs in SCHEMA testnm TO readonly;
GRANT

testdb=#
```

###### ✅ 12 Создайте пользователя testread с паролем test123

```
testdb=# CREATE USER testread with password 'test123';
CREATE ROLE

testdb=#
```

###### ✅ 13 Дайте роль readonly пользователю testread

```
testdb=# grant readonly TO testread;
GRANT ROLE

testdb=#
```

###### ✅ 14 Зайдите под пользователем testread в базу данных testdb

```
root@otus:~# psql -h 127.0.0.1 -U testread -d testdb -W
Password:
psql (14.8 (Ubuntu 14.8-1.pgdg22.04+1))
SSL connection (protocol: TLSv1.3, cipher: TLS_AES_256_GCM_SHA384, bits: 256, compression: off)
Type "help" for help.

testdb=>
```

###### ✅ 15 Сделайте select * from t1;

```
ERROR:  relation "t1" does not exist
LINE 1: SELECT * FROM t1;
                      ^
testdb=>
```

###### ✅ 16 Получилось? (могло если вы делали сами не по шпаргалке и не упустили один существенный момент про который позже)
###### ✅ 17 Напишите что именно произошло в тексте домашнего задания
###### ✅ 18 У вас есть идеи почему? ведь права то дали?

При запросе вида `select * from t1;` без прямого указания схемы 'testnm' получение данных будет невозможно или запрещено, т. к. по-умолчанию путь поиска схемы идет в порядке 'имя текущего пользователя' -> схема 'public', в которых таблица 't1' не существует. 

На шаге №6 таблица была создана в схеме 'testnm', делаем соответствующий запрос и видим данные:

```
testdb=> SELECT * FROM testnm.t1;
 c1
----
(0 rows)

testdb=>
```



###### ✅ 19 Посмотрите на список таблиц 

Таблица 't1' в схеме 'testnm':

```
testdb-# \dt+
                                     List of relations
 Schema | Name | Type  |  Owner   | Persistence | Access method |    Size    | Description
--------+------+-------+----------+-------------+---------------+------------+-------------
 testnm | t1   | table | postgres | permanent   | heap          | 8192 bytes |
(1 row)

testdb-#
```


Шаги 20-28 пропущу, т. к. изначально создано корректно:

	20 подсказка в шпаргалке под пунктом 20  
	21 а почему так получилось с таблицей (если делали сами и без шпаргалки то может у вас все нормально)  
	22 вернитесь в базу данных testdb под пользователем postgres  
	23 удалите таблицу t1  
	24 создайте ее заново но уже с явным указанием имени схемы testnm  
	25 вставьте строку со значением c1=1  
	26 зайдите под пользователем testread в базу данных testdb  
	27 сделайте select * from testnm.t1;  
	28 получилось? 


###### ✅ 29 Есть идеи почему? если нет - смотрите шпаргалку

В таком случае очевидно доступ будет запрещен - право на чтение будет утрачено из-за пересоздания таблицы.


###### ✅ 30 Как сделать так чтобы такое больше не повторялось? если нет идей - смотрите шпаргалку

Необходимо определить права доступа по умолчанию для вновь созданных таблиц в схеме 'testnm' (команду взял из подсказок):
https://postgrespro.ru/docs/postgresql/9.6/sql-alterdefaultprivileges

```
postgres=# \c testdb postgres;
You are now connected to database "testdb" as user "postgres".
testdb=# ALTER default privileges in SCHEMA testnm grant SELECT on TABLEs to readonly;
ALTER DEFAULT PRIVILEGES

testdb=#
```



###### ✅ 31 Сделайте select * from testnm.t1;

```
testdb=> \c testdb;
testdb=> select * from testnm.t1;
 c1
----
  1
(1 row)

testdb=>
```

###### ✅ 32 Получилось? 

В моем сценарии да

###### ✅ 33 Есть идеи почему? если нет - смотрите шпаргалку

Полностью согласен с комментарием в подсказке

###### ✅ 31 Сделайте select * from testnm.t1;
###### ✅ 32 Получилось?  
###### ✅ 33 Ура! 


###### ✅ 34 Теперь попробуйте выполнить команду create table t2(c1 integer); insert into t2 values (2);

Запрос успешно выполнится:
```
testdb=> create table t2(c1 integer); insert into t2 values (2);
CREATE TABLE
INSERT 0 1

testdb=>
```

###### ✅ 35 А как так? нам же никто прав на создание таблиц и insert в них под ролью readonly? 

При создании пользователей им по умолчанию добавляется роль 'public', которая позволяет делать запись в схеме 'public', а запрос на шаге №34 как раз и создает таблицу 't2' в этой схеме.


###### ✅ 36 Есть идеи как убрать эти права? если нет - смотрите шпаргалку 

Первое что приходит на ум - отозвать все привелегии у роли 'public'
https://postgrespro.ru/docs/postgrespro/10/sql-revoke

В подсказке рекомендуются команды:

```
testdb=# revoke CREATE on SCHEMA public FROM public;
REVOKE
testdb=# revoke all on DATABASE testdb FROM public;
REVOKE

testdb=#
```

Как я их понимаю:

1. Команда `REVOKE CREATE ON SCHEMA public FROM public;` отзывает привилегию на создание объектов в рамках схемы 'public' у роли 'public', соответственно пользователи с ролью 'public теперь не смогут создавать объекты в схеме 'public';

2. Команда `REVOKE ALL ON DATABASE testdb FROM public;` отзывает все привилегии у роли 'public' в рамках базы данных 'testdb' - удаляются вообще все привилегии, по сути, эта команда запрещает любой доступ или разрешения для пользователей с ролью 'public' к базе данных 'testdb';


###### ✅ 37 если вы справились сами то расскажите что сделали и почему, если смотрели шпаргалку - объясните что сделали и почему выполнив указанные в ней команды  

Обратился к подсказке и разобрался как работают команды. Спасибо!

###### ✅ 38 теперь попробуйте выполнить команду create table t3(c1 integer); insert into t2 values (2);  

Выполнено:
```
testdb=> create table t3(c1 integer); insert into t2 values (2);
ERROR:  permission denied for schema public
```


###### ✅ 39 расскажите что получилось и почему

Теперь пользователь 'testread' не имеет разрешений для манипуляций в схеме 'public', соответственно видим ошибку доступа.
