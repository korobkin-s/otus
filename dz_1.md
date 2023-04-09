## Домашнее задание №1

#### Работа с уровнями изоляции транзакции в PostgreSQL

Цель:

-   научиться работать с Google Cloud Platform на уровне Google Compute Engine (IaaS)
-   научиться управлять уровнем изоляции транзации в PostgreSQL и понимать особенность работы уровней read commited и repeatable read
  

### Описание/Пошаговая инструкция выполнения домашнего задания:

 ###### 1. Создать новый проект в Google Cloud Platform, Яндекс облако или на любых ВМ, докере

	✅ Для проекта выбрана облачная платформа Yandex Cloud;

###### 2. Создать инстанс виртуальной машины с дефолтными параметрами

	✅ OS Ubuntu 22.04 LTS/2 vCPU/2GB RAM/SSD 15 ГБ;

###### 3. Добавить свой ssh ключ в metadata ВМ 
	✅ Выполнено;
	
###### 4. Зайти удаленным ssh (первая сессия), не забывайте про ssh-add 
	✅ Выполнено;
	
###### 5. Поставить PostgreSQL
	✅ Установлен PostgreSQL 15.2;
	
###### 6. Зайти вторым ssh (вторая сессия)
	 ✅ Выполнено;

###### 7. Запустить везде psql из под пользователя postgres
	✅ Выполнено;
	
###### 8. Выключить auto commit:
	postgres=# \set AUTOCOMMIT OFF
	postgres=# \echo :AUTOCOMMIT
	OFF
	postgres=#
	
###### 9. Сделать в первой сессии новую таблицу и наполнить ее данными:

```
postgres=# create table persons(id serial, first_name text, second_name text);
        insert into persons(first_name, second_name) values('ivan', 'ivanov');
        insert into persons(first_name, second_name) values('petr', 'petrov');
        commit;
CREATE TABLE
INSERT 0 1
INSERT 0 1
COMMIT

postgres=#
```

###### 10. Посмотреть текущий уровень изоляции
	postgres=# show transaction isolation level;
	 transaction_isolation
	-----------------------
	 read committed
	(1 row)
	
	postgres=*#

###### 11. Начать новую транзакцию в обоих сессиях с дефолтным (не меняя) уровнем изоляции. В первой сессии добавить новую запись
```
postgres=# insert into persons(first_name, second_name) values('sergey', 'sergeev');
INSERT 0 1

postgres=*#
```
###### 12. Сделать `select * from persons` во второй сессии. Видите ли вы новую запись и если да то почему?

```
postgres=# select * from persons;
 id | first_name | second_name
----+------------+-------------
  1 | ivan       | ivanov
  2 | petr       | petrov
(2 rows)

postgres=*#
```

❓ _**Ответ:** второй сеанс не сможет увидеть новую запись, пока транзация в первом сеансе не будет зафиксирована_


###### 13. Завершить первую транзакцию
```
postgres=*# commit;
COMMIT

postgres=#
```
###### 14. Сделать `select * from persons` во второй сессии. Видите ли вы новую запись и если да то почему?

```
postgres=# select * from persons;
 id | first_name | second_name
----+------------+-------------
  1 | ivan       | ivanov
  2 | petr       | petrov
  4 | sergey     | sergeev
(3 rows)

postgres=*#
```

❓ _**Ответ:** да, т.к. транзация в первом сеансе зафиксирована_

###### 15. Завершите транзакцию во второй сессии
```
postgres=*# commit;
COMMIT

postgres=#
```
###### 16. Начать новые но уже repeatable read транзации - `set transaction isolation level repeatable read;`

```
postgres=# set transaction isolation level repeatable read;
SET
postgres=*# show transaction isolation level;
 transaction_isolation
-----------------------
 repeatable read
(1 row)

postgres=*#
```

###### 17. В первой сессии добавить новую запись `insert into persons(first_name, second_name) values('sveta', 'svetova');`
```
postgres=*# insert into persons(first_name, second_name) values('sveta', 'svetova');
INSERT 0 1

postgres=*#
```
	
###### 18. Сделать `select * from persons;` во второй сессии. Видите ли вы новую запись и если да то почему? 

	postgres=*# select * from persons;
	 id | first_name | second_name
	----+------------+-------------
	  1 | ivan       | ivanov
	  2 | petr       | petrov
	  4 | sergey     | sergeev
	(3 rows)
	
	postgres=*#

❓ _**Ответ:** нет, новая запись не будет отображаться до тех пор, пока не будут зафиксированы транзации в обеих сессиях_


###### 19. Завершить первую транзакцию - commit;

```
postgres=*# commit;
COMMIT

postgres=#
```

###### 20. Сделать `select * from persons` во второй сессии. Видите ли вы новую запись и если да то почему?

```
postgres=*# select * from persons;
 id | first_name | second_name
----+------------+-------------
  1 | ivan       | ivanov
  2 | petr       | petrov
  4 | sergey     | sergeev
(3 rows)

postgres=*#
```

❓ _**Ответ:** второй сеанс все еще видит тот же снимок базы данных, что и в начале своей транзакции, соответственно новая запись видна не будет_

###### 21. Завершить вторую транзакцию

```
postgres=*# commit;
COMMIT

postgres=#
```

###### 22. Сделать select * from persons во второй сессии. Видите ли вы новую запись и если да то почему?

	postgres=# select * from persons;
	 id | first_name | second_name
	----+------------+-------------
	  1 | ivan       | ivanov
	  2 | petr       | petrov
	  4 | sergey     | sergeev
	  5 | sveta      | svetova
	(4 rows)
	
	postgres=*#

❓ _**Ответ:** да, новая запись будет видна, т.к. теперь транзация во втором сенсе видит актуальный снимок базы_
