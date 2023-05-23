## Настройка autovacuum с учетом особеностей производительности

Цель:

-   запустить нагрузочный тест pgbench
-   настроить параметры autovacuum
-   проверить работу autovacuum


### Описание/Пошаговая инструкция выполнения домашнего задания:

###### ✅ Создать инстанс ВМ с 2 ядрами и 4 Гб ОЗУ и SSD 10GB

	Для проекта выбрана облачная платформа Yandex Cloud;
	VM: OS Ubuntu 22.04 LTS/2 vCPU/4GB RAM/SSD 18 

###### ✅ Установить на него PostgreSQL 15 с дефолтными настройками

```
root@otus:~# pg_config --version
PostgreSQL 15.3 (Ubuntu 15.3-1.pgdg22.04+1)

root@otus:~#
```

###### ✅ Создать БД для тестов: выполнить pgbench -i postgres

Создана БД 'testdb' и выполнена ее подготовка для тестирования средствами pgbench:

```
postgres=# CREATE DATABASE testdb;
CREATE DATABASE
postgres=#


root@otus:~# pgbench -h localhost -U postgres -i testdb
Password:
dropping old tables...
NOTICE:  table "pgbench_accounts" does not exist, skipping
NOTICE:  table "pgbench_branches" does not exist, skipping
NOTICE:  table "pgbench_history" does not exist, skipping
NOTICE:  table "pgbench_tellers" does not exist, skipping
creating tables...
generating data (client-side)...
100000 of 100000 tuples (100%) done (elapsed 0.03 s, remaining 0.00 s)
vacuuming...
creating primary keys...
done in 1.00 s (drop tables 0.00 s, create tables 0.02 s, client-side generate 0.67 s, vacuum 0.04 s, primary keys 0.27 s).

root@otus:~#
```


###### ✅ Запустить pgbench -c8 -P 6 -T 60 -U postgres postgres

Запущен тест pgbench с указанными параметрами:
```
root@otus:~# pgbench -h localhost -c8 -P 6 -T 60 -U postgres testdb
Password:
pgbench (15.3 (Ubuntu 15.3-1.pgdg22.04+1))
starting vacuum...end.
progress: 6.0 s, 357.7 tps, lat 21.954 ms stddev 20.424, 0 failed
progress: 12.0 s, 437.0 tps, lat 18.337 ms stddev 24.499, 0 failed
progress: 18.0 s, 322.8 tps, lat 24.778 ms stddev 22.235, 0 failed
progress: 24.0 s, 482.2 tps, lat 16.585 ms stddev 57.964, 0 failed
progress: 30.0 s, 612.3 tps, lat 13.066 ms stddev 11.573, 0 failed
progress: 36.0 s, 476.7 tps, lat 16.778 ms stddev 13.986, 0 failed
progress: 42.0 s, 562.2 tps, lat 14.236 ms stddev 51.207, 0 failed
progress: 48.0 s, 480.8 tps, lat 16.620 ms stddev 18.531, 0 failed
progress: 54.0 s, 536.0 tps, lat 14.936 ms stddev 12.738, 0 failed
progress: 60.0 s, 550.3 tps, lat 14.525 ms stddev 12.419, 0 failed
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 8
number of threads: 1
maximum number of tries: 1
duration: 60 s
number of transactions actually processed: 28916
number of failed transactions: 0 (0.000%)
latency average = 16.576 ms
latency stddev = 29.610 ms
initial connection time = 96.318 ms
tps = 482.527212 (without initial connection time)

root@otus:~#
```

###### ✅ Применить параметры настройки PostgreSQL из прикрепленного к материалам занятия файла

Настройки в postgresql.conf приведены к следующему виду:

```
max_connections = 40
shared_buffers = 1GB
effective_cache_size = 3GB
maintenance_work_mem = 512MB
checkpoint_completion_target = 0.9
wal_buffers = 16MB
default_statistics_target = 500
random_page_cost = 4
effective_io_concurrency = 2
work_mem = 6553kB
min_wal_size = 4GB
max_wal_size = 16GB
```


###### ✅ Протестировать заново

Координально изменилось значение 'latency stddev':

```
root@otus:~# pgbench -h localhost -c8 -P 6 -T 60 -U postgres testdb
Password:
pgbench (15.3 (Ubuntu 15.3-1.pgdg22.04+1))
starting vacuum...end.
progress: 6.0 s, 552.2 tps, lat 14.227 ms stddev 11.953, 0 failed
progress: 12.0 s, 567.3 tps, lat 14.100 ms stddev 8.680, 0 failed
progress: 18.0 s, 534.5 tps, lat 14.939 ms stddev 10.708, 0 failed
progress: 24.0 s, 492.0 tps, lat 16.278 ms stddev 13.008, 0 failed
progress: 30.0 s, 570.3 tps, lat 13.994 ms stddev 12.743, 0 failed
progress: 36.0 s, 376.7 tps, lat 21.300 ms stddev 20.721, 0 failed
progress: 42.0 s, 356.8 tps, lat 22.310 ms stddev 15.210, 0 failed
progress: 48.0 s, 463.5 tps, lat 17.295 ms stddev 14.715, 0 failed
progress: 54.0 s, 372.8 tps, lat 21.519 ms stddev 20.526, 0 failed
progress: 60.0 s, 476.7 tps, lat 16.676 ms stddev 17.151, 0 failed
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 8
number of threads: 1
maximum number of tries: 1
duration: 60 s
number of transactions actually processed: 28585
number of failed transactions: 0 (0.000%)
latency average = 16.770 ms
latency stddev = 14.828 ms
initial connection time = 102.619 ms
tps = 476.812922 (without initial connection time)

root@otus:~#
```



###### ✅ Что изменилось и почему?

Количество транзакций в секунду (tps) в целом осталось на прежнем уровне, но произошло двухкратное уменьшение итоговых значений параметра 'latency stddev', который отражает стандартное время (разброс во времени завершения транзакций) задержки транзакций в миллисекундах.

Потенциально сократить эту задержку помогло увеличение значений таких параметров как 'shared_buffers', 'effective_cache_size' и 'work_mem' - использование кэша и памяти большего объема способствовало более эффективному использованию ресурсов и как следствие сказалось на уменьшении задержек и повышении общей производительности.


###### ✅ Создать таблицу с текстовым полем и заполнить случайными или сгенерированными данным в размере 1млн строк

```
postgres=# \c testdb;
You are now connected to database "testdb" as user "postgres".
testdb=# CREATE TABLE student(
  id serial,
  fio char(100)
);
CREATE INDEX idx_fio ON student(fio);
CREATE TABLE
CREATE INDEX
testdb=#
testdb=#
testdb=#
testdb=# INSERT INTO student(fio) SELECT 'noname' FROM generate_series(1,1000000);
INSERT 0 1000000

testdb=#
```



###### ✅ Посмотреть размер файла с таблицей

Размер файла 142 MB:

```
testdb=# SELECT pg_size_pretty(pg_total_relation_size('student'));
 pg_size_pretty
----------------
 142 MB
(1 row)

testdb=#
```


###### ✅ 5 раз обновить все строчки и добавить к каждой строчке любой символ

```
testdb=# WITH RECURSIVE update_student AS (
  SELECT 1 AS iteration
  UNION ALL
  SELECT iteration + 1
  FROM update_student
  WHERE iteration < 5
)
UPDATE student
SET fio = 'name'
FROM update_student;
UPDATE 1000000

testdb=#
```

###### ✅ Посмотреть количество мертвых строчек в таблице и когда последний раз приходил автовакуум  

```
testdb=# SELECT relname, n_live_tup, n_dead_tup, trunc(100*n_dead_tup/(n_live_tup+1))::float "ratio%", last_autovacuum FROM pg_stat_user_TABLEs WHERE relname = 'student';
 relname | n_live_tup | n_dead_tup | ratio% |        last_autovacuum
---------+------------+------------+--------+-------------------------------
 student |    1000000 |    1000000 |     99 | 2023-05-23 19:13:30.067699+00
(1 row)

testdb=#
```


###### ✅ Подождать некоторое время, проверяя, пришел ли автовакуум

Автовакуум отработал:
```
testdb=# SELECT relname, n_live_tup, n_dead_tup, trunc(100*n_dead_tup/(n_live_tup+1))::float "ratio%", last_autovacuum FROM pg_stat_user_TABLEs WHERE relname = 'student';
 relname | n_live_tup | n_dead_tup | ratio% |        last_autovacuum
---------+------------+------------+--------+-------------------------------
 student |    1000000 |          0 |      0 | 2023-05-23 19:15:29.825139+00
(1 row)

testdb=#
```

###### ✅ 5 раз обновить все строчки и добавить к каждой строчке любой символ

Выполнено:
```
testdb=# WITH RECURSIVE update_student AS (
  SELECT 1 AS iteration
  UNION ALL
  SELECT iteration + 1
  FROM update_student
  WHERE iteration < 5
)
UPDATE student
SET fio = 'name_1'
FROM update_student;
UPDATE 1000000
testdb=#
testdb=#
testdb=# SELECT relname, n_live_tup, n_dead_tup, trunc(100*n_dead_tup/(n_live_tup+1))::float "ratio%", last_autovacuum FROM pg_stat_user_TABLEs WHERE relname = 'student';
 relname | n_live_tup | n_dead_tup | ratio% |        last_autovacuum
---------+------------+------------+--------+-------------------------------
 student |    1000000 |    1000000 |     99 | 2023-05-23 19:15:29.825139+00
(1 row)

testdb=#
```



###### ✅ Посмотреть размер файла с таблицей

Размер увеличился примерно в 2 раза:

```
testdb=# SELECT pg_size_pretty(pg_total_relation_size('student'));
 pg_size_pretty
----------------
 296 MB
(1 row)

testdb=#
```

###### ✅ Отключить Автовакуум на конкретной таблице

Выполнено:
```
testdb=# ALTER TABLE student SET (autovacuum_enabled = off);
ALTER TABLE
testdb=#
```

###### ✅ 10 раз обновить все строчки и добавить к каждой строчке любой символ

```
testdb=# WITH RECURSIVE update_student AS (
  SELECT 1 AS iteration
  UNION ALL
  SELECT iteration + 1
  FROM update_student
  WHERE iteration < 10
)
UPDATE student
SET fio = 'name_2'
FROM update_student;
UPDATE 1000000

testdb=#
```

###### ✅ Посмотреть размер файла с таблицей  

Размер ожидаемо начал увеличиваться:
```
testdb=# SELECT pg_size_pretty(pg_total_relation_size('student'));
 pg_size_pretty
----------------
 444 MB
(1 row)

testdb=#
```
###### ✅ Объясните полученный результат 

Отключение Autovacuum приводит к накоплению неиспользуемых данных и увеличению размера файла таблицы в PostgreSQL - после обновления строк в таблице они помечаются как "мёртвые" и занимают место + после обновления данных в таблице также должны быть обновлены индексы, а если Autovacuum выключен, индексы не будут оптимизированы автоматически, а значит будут также занимать больше места.


###### ✅ Написать анонимную процедуру, в которой в цикле 10 раз обновятся все строчки в искомой таблице.

Пример процедуры:

```
testdb=# DO $$
DECLARE
   step_counter INT := 1;
BEGIN
   FOR i IN 1..10 LOOP
      UPDATE student SET fio = 'name_3';
      RAISE NOTICE 'Шаг %: Обновлено поле fio в таблице student для всех записей', step_counter;
      step_counter := step_counter + 1;
   END LOOP;
END $$;
NOTICE:  Шаг 1: Обновлено поле fio в таблице student для всех записей
NOTICE:  Шаг 2: Обновлено поле fio в таблице student для всех записей
NOTICE:  Шаг 3: Обновлено поле fio в таблице student для всех записей
NOTICE:  Шаг 4: Обновлено поле fio в таблице student для всех записей
NOTICE:  Шаг 5: Обновлено поле fio в таблице student для всех записей
NOTICE:  Шаг 6: Обновлено поле fio в таблице student для всех записей
NOTICE:  Шаг 7: Обновлено поле fio в таблице student для всех записей
NOTICE:  Шаг 8: Обновлено поле fio в таблице student для всех записей
NOTICE:  Шаг 9: Обновлено поле fio в таблице student для всех записей
NOTICE:  Шаг 10: Обновлено поле fio в таблице student для всех записей
DO

testdb=#
```
