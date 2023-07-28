### Бэкапы

Цель:
Применить логический бэкап. Восстановиться из бэкапа



###### ✅ 1. Создаем ВМ/докер c ПГ.

Для проекта выбрана облачная платформа Yandex Cloud

	VM: OS Ubuntu 22.04 LTS/2 vCPU/2GB RAM/SSD 18
 
	СУБД: PostgreSQL 15.3
	

###### ✅ 2. Создаем БД, схему и в ней таблицу.

Создаем базу 'otus' со схемой 'security' и две таблицы - 'table_1' и 'table_2':

```
root@otus:~# sudo -u postgres psql
psql (15.3 (Ubuntu 15.3-1.pgdg22.04+1))
Type "help" for help.

CREATE DATABASE
postgres=# \c otus 
You are now connected to database "otus" as user "postgres".

otus=# create schema security;
CREATE SCHEMA

otus=# create table security.table_1 (password char(100));
CREATE TABLE

otus=# create table security.table_2 (password char(100));
CREATE TABLE

otus=#
```

###### ✅ 3. Заполним таблицы автосгенерированными 100 записями.

Заполняем таблицу 'table_1' и проверяем, что данные есть:

```
otus=# insert into security.table_1 (password)
select md5(random()::text)
from generate_series(1, 100);
INSERT 0 100

otus=# select * from security.table_1 limit 5;
                                               password                                               
------------------------------------------------------------------------------------------------------
 e8e6d6ab504b82de04c1034736d2a117                                                                    
 3523894bf3820e81cabac3e1b74ac63e                                                                    
 09b9816874f0ae15dcfa449fa4ec7971                                                                    
 5e9daccd94fdbc92b889d0d3415139a7                                                                    
 e553852f2251567caca4ac956bb49511                                                                    
(5 rows)

otus=#
```

Заполняем таблицу 'table_2':

```
otus=# insert into security.table_2 (password)
select md5(random()::text)
from generate_series(1, 100);
INSERT 0 100

otus=# select * from security.table_2 limit 5;
                                               password                                               
------------------------------------------------------------------------------------------------------
 8792b1201959dc3529b1830f2aeb5896                                                                    
 691c5771880dc1de4ff1ab6c893a6c9e                                                                    
 9993dbbeab3e8d0d4473f6ec431abce2                                                                    
 23847dc72e5f367326adf51c20cdb581                                                                    
 5356b6cc092db5fe15de7ad9aead69d6                                                                    
(5 rows)

otus=#
```


###### ✅ 4. Под линукс пользователем Postgres создадим каталог для бэкапов

Предположим мы хотим, чтобы бэкапы распологались в директории '/mnt', но у пользователя 'postgres' изначально не будет прав на создание объектов в этом расположении. От имени суперпользователя создаем там папку 'backup' и назначаем пользователя 'postgres' владельцем:

```
root@otus:~# mkdir /mnt/backup

root@otus:~# chown postgres:postgres /mnt/backup
```

###### ✅ 5. Сделаем логический бэкап используя утилиту COPY


С помощью команды '\copy' экспортируем данные из таблицы 'security.table_1' в файл 'otus.security.table_1.sql':

```
otus=# \copy security.table_1 to '/mnt/backup/otus.security.table_1.sql';
COPY 100

otus=#
```

###### ✅ 6. Восстановим во вторую таблицу данные из бэкапа.

Экспортируем данные из бэкапа в таблицу 'security.table_2':

```
otus=# \copy security.table_2 from '/mnt/backup/otus.security.table_1.sql';
```

И проверяем количество строк в таблице (+100):

```
otus=# select count(*) as count from security.table_2;
 count 
-------
   200
(1 row)

otus=#
```


###### ✅ 7. Используя утилиту pg_dump создадим бэкап в кастомном сжатом формате двух таблиц


Переключаемся на пользователя 'postgres', для создания бэкапа определенных таблиц используем опцию '-t':

```
root@otus:~# su - postgres
postgres@otus:~$ pg_dump -d otus -t security.table_1 -t security.table_2 -Fc -f /mnt/backup/otus_tables.backup

postgres@otus:~$
```


Для создания полного бэкапа команда будет выглядеть так:

```
postgres@otus:~$ pg_dump -d otus -Fc -f /mnt/backup/otus.backup
```


###### ✅ 8. Используя утилиту pg_restore восстановим в новую БД только вторую таблицу!  


Предварительно создадим новую базу 'otus_new' с аналогичной схемой и таблицей:

```
postgres=# create database otus_new;
CREATE DATABASE

postgres=# \c otus_new 
You are now connected to database "otus_new" as user "postgres".

otus_new=# create schema security;
CREATE SCHEMA

otus_new=# create table security.table_2 (password char(100));
CREATE TABLE

otus_new=#
```


Восстанавливаем с непосредственным указанием таблицы назначения и схемы, используем опцию '--data-only' для восстановления только данных:

```
root@otus:~# su - postgres
postgres@otus:~$ pg_restore -d otus_new -t table_2 -n security --data-only -v /mnt/backup/otus_tables.backup
pg_restore: connecting to database for restore
pg_restore: processing data for table "security.table_2"

postgres@otus:~$
```


Проверяем содержимое и количество строк (состояние данных идентично данным на шаге №6):


```
otus_new=# select * from security.table_2 limit 5;
                                               password                                               
------------------------------------------------------------------------------------------------------
 8792b1201959dc3529b1830f2aeb5896                                                                    
 691c5771880dc1de4ff1ab6c893a6c9e                                                                    
 9993dbbeab3e8d0d4473f6ec431abce2                                                                    
 23847dc72e5f367326adf51c20cdb581                                                                    
 5356b6cc092db5fe15de7ad9aead69d6                                                                    
(5 rows)

postgres=# \c otus_new 
You are now connected to database "otus_new" as user "postgres".
otus_new=# select count(*) as count from security.table_2;
 count 
-------
   200
(1 row)

otus_new=#
```


В моем случае таблица не содержала первичный ключ и/или ограничения уникальности, и восстановление прогнозируемо прошло без ошибок.

Если же целевая таблица будет содержать такие элементы, то при попытке восстановления произойдет ошибка из-за конфликтов. В таком случае, предварительно потребуется очистить содержимое этой таблицы командой `DELETE FROM` или `TRUNCATE TABLE` , не забыв перед этим обязательно сделать полный бэкап исходной БД или конкретной таблицы.
