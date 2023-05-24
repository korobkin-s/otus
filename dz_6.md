## Механизм блокировок

Цель:

-   понимать как работает механизм блокировок объектов и строк


Описание/Пошаговая инструкция выполнения домашнего задания:

#### ✅ 1.  Настройте сервер так, чтобы в журнал сообщений сбрасывалась информация о блокировках, удерживаемых более 200 миллисекунд. Воспроизведите ситуацию, при которой в журнале появятся такие сообщения.

Задаем 'deadlock_timeout' = 200 ms:

```
postgres=# SHOW deadlock_timeout;
 deadlock_timeout
------------------
 200ms
(1 row)

postgres=#
```

Запускаем выполнение транзации в первой сессии:

```
postgres=#\c locks
You are now connected to database "locks" as user "postgres".
locks=#
locks=#
locks=# BEGIN;
UPDATE accounts SET amount = amount - 100.00 WHERE acc_no = 1;
BEGIN
UPDATE 1
locks=*#

```

И затем во второй сессии:

```
postgres=# \c locks
You are now connected to database "locks" as user "postgres".
locks=# BEGIN;
UPDATE accounts SET amount = amount + 100.00 WHERE acc_no = 1;
BEGIN
UPDATE 1
locks=*# COMMIT;
COMMIT
locks=#
```

Спустя некоторое время фиксируем транзации в первой сессии и затем во второй:

```
locks=*# COMMIT;
COMMIT
locks=#
```

В логах зафиксировано выполнение транзации из второй сессии спустя 10,5 сек ожидания:

```
root@otus:~# tail -n 10 /var/log/postgresql/postgresql-15-main.log
2023-05-24 16:47:26.287 UTC [55188] LOG:  checkpoint starting: time
2023-05-24 16:47:26.301 UTC [55188] LOG:  checkpoint complete: wrote 3 buffers (0.0%); 0 WAL file(s) added, 0 removed, 0 recycled; write=0.004 s, sync=0.002 s, total=0.014 s; sync files=2, longest=0.001 s, average=0.001 s; distance=0 kB, estimate=0 kB
2023-05-24 16:49:52.293 UTC [55304] postgres@locks LOG:  process 55304 still waiting for ShareLock on transaction 738 after 200.094 ms
2023-05-24 16:49:52.293 UTC [55304] postgres@locks DETAIL:  Process holding the lock: 55302. Wait queue: 55304.
2023-05-24 16:49:52.293 UTC [55304] postgres@locks CONTEXT:  while updating tuple (0,1) in relation "accounts"
2023-05-24 16:49:52.293 UTC [55304] postgres@locks STATEMENT:  UPDATE accounts SET amount = amount + 100.00 WHERE acc_no = 1;
2023-05-24 16:50:02.611 UTC [55304] postgres@locks LOG:  process 55304 acquired ShareLock on transaction 738 after 10518.357 ms
2023-05-24 16:50:02.611 UTC [55304] postgres@locks CONTEXT:  while updating tuple (0,1) in relation "accounts"
2023-05-24 16:50:02.611 UTC [55304] postgres@locks STATEMENT:  UPDATE accounts SET amount = amount + 100.00 WHERE acc_no = 1;
root@otus:~#
```


#### ✅ 2.  Смоделируйте ситуацию обновления одной и той же строки тремя командами UPDATE в разных сеансах. Изучите возникшие блокировки в представлении pg_locks и убедитесь, что все они понятны. Пришлите список блокировок и объясните, что значит каждая.

###### В первой сессии:
1. Смотрим идентификатор процесса сервера, к которому подключен текущий сеанс;
2. Обновляем значение 'amount' в первой строке (acc_no = 1);
3. Смотрим вывод из pg_locks для полученного pid=55571:
	1. В первой строке сообщается, что процесс 55571 удерживает эксклюзивную блокировку на таблицу"accounts" в режиме RowExclusiveLock, т. е. другие процессы не могут получить эксклюзивную блокировку на данную таблицу, пока эта блокировка не будет освобождена. Поле "granted" имеет значение "t" - блокировка для этого pid получена;
	2. Во второй строке сообщается, что процесс 55571 также удерживает эксклюзивную блокировку на транзакционный идентификатор 751 (текущего сеанса) в режиме ExclusiveLock, т. е. другие процессы не могут получить эксклюзивную блокировку на этот конкретный идентификатор транзакции до освобождения этой блокировки. Поле "granted" также имеет значение "t" - блокировка для этого pid получена;

```
postgres=# BEGIN;
SELECT txid_current(), pg_backend_pid();
BEGIN
 txid_current | pg_backend_pid
--------------+----------------
          751 |          55571
(1 row)

postgres=*# UPDATE accounts SET amount = amount + 100.00 WHERE acc_no = 1;
SELECT * FROM locks_v WHERE pid = 638;
UPDATE 1
 pid | locktype | lockid | mode | granted
-----+----------+--------+------+---------
(0 rows)

postgres=*# SELECT * FROM locks_v WHERE pid = 55571;
  pid  |   locktype    |  lockid  |       mode       | granted
-------+---------------+----------+------------------+---------
 55571 | relation      | accounts | RowExclusiveLock | t
 55571 | transactionid | 751      | ExclusiveLock    | t
(2 rows)
```

###### Во второй сессии:

1. Смотрим идентификатор процесса сервера, к которому подключен текущий сеанс;
2. Запускаем обновление значения 'amount' в первой строке (acc_no = 1);
3. Смотрим вывод из pg_locks для полученного pid=55570:
	1. В первой строке сообщается, что процесс 55570 удерживает эксклюзивную блокировку на таблицу "accounts" в режиме RowExclusiveLock, т. е. другие процессы не могут получить эксклюзивную блокировку на данную таблицу, пока эта блокировка не будет освобождена. Поле "granted" имеет значение "t" - блокировка для этого pid получена;
	2. Во второй строке сообщается, что процесс 55570 удерживает эксклюзивную блокировку на строку с идентификатором "accounts:1" в таблице "accounts" в режиме ExclusiveLock, аналогично другие процессы не могут получить эксклюзивную блокировку на этоту строку до освобождения этой блокировки. Поле "granted" имеет значение "t" - блокировка для этого pid получена;
	3. В третье строке сообщается, что процесс 55570 удерживает эксклюзивную блокировку на транзакционный идентификатор 752 (текущего сеанса) в режиме ExclusiveLock, т. е. другие процессы не могут получить эксклюзивную блокировку на этот идентификатор транзакции до освобождения этой блокировки. Поле "granted" имеет значение "t" - блокировка для этого pid получена;
	4. В четвертой строке сообщается, что процесс 55570 удерживает разделяемую блокировку (ShareLock) на транзакционный идентификатор 751 из первого сеанса. Поле "granted" имеет значение "f" -  блокировка этому процессу не была предоставлена и находится в ожидании;

```
postgres=# BEGIN;
SELECT txid_current(), pg_backend_pid();
UPDATE accounts SET amount = amount + 100.00 WHERE acc_no = 1;
BEGIN
 txid_current | pg_backend_pid
--------------+----------------
          752 |          55570
(1 row)


wait...
```

```
postgres=*# SELECT * FROM locks_v WHERE pid = 55570;
  pid  |   locktype    |   lockid   |       mode       | granted
-------+---------------+------------+------------------+---------
 55570 | relation      | accounts   | RowExclusiveLock | t
 55570 | tuple         | accounts:1 | ExclusiveLock    | t
 55570 | transactionid | 752        | ExclusiveLock    | t
 55570 | transactionid | 751        | ShareLock        | f
(4 rows)
```


###### В третьей сессии:

1. Смотрим идентификатор процесса сервера, к которому подключен текущий сеанс;
2. Запускаем обновление значения 'amount' в первой строке (acc_no = 1);
3. Смотрим вывод из pg_locks для полученного pid=55569:
	1. В первой строке сообщается, что процесс 55569 удерживает эксклюзивную блокировку на таблицу с именем "accounts" в режиме RowExclusiveLock, т. е. другие процессы не могут получить эксклюзивную блокировку на данную таблицу, пока эта блокировка не будет освобождена. Поле "granted" имеет значение "t" - блокировка для этого pid получена;
	2. Во второй строке сообщается, что процесс 55569 пытался получить эксклюзивную блокировку на строку с идентификатором "accounts:1" в таблице "accounts" в режиме ExclusiveLock, но не был ей предоставлен. Значение поля "granted" имеет значение "f" и указывает на неудачу получения этой блокировки.
	3. Во второй строке сообщается, что процесс 55569 удерживает эксклюзивную блокировку на транзакционный идентификатор 753 (текущего сеанса) в режиме ExclusiveLock, т. е. другие процессы не могут получить эксклюзивную блокировку на этот идентификатор транзакции до освобождения этой блокировки. Поле "granted" имеет значение "t" - блокировка для этого pid получена;

```
postgres=# BEGIN;
SELECT txid_current(), pg_backend_pid();
UPDATE accounts SET amount = amount + 100.00 WHERE acc_no = 1;
BEGIN
 txid_current | pg_backend_pid
--------------+----------------
          753 |          55569
(1 row)


wait...
```

```
postgres=*# SELECT * FROM locks_v WHERE pid = 55569;
  pid  |   locktype    |   lockid   |       mode       | granted
-------+---------------+------------+------------------+---------
 55569 | relation      | accounts   | RowExclusiveLock | t
 55569 | tuple         | accounts:1 | ExclusiveLock    | f
 55569 | transactionid | 753        | ExclusiveLock    | t
(3 rows)
```


#### ✅ 3.  Воспроизведите взаимоблокировку трех транзакций. Можно ли разобраться в ситуации постфактум, изучая журнал сообщений?

##### Сессия №1

Обновляем значение 'amount' в первой строке:
```
locks=# BEGIN;
SELECT txid_current(), pg_backend_pid();
UPDATE accounts SET amount = amount - 100.00 WHERE acc_no = 1;
BEGIN
 txid_current | pg_backend_pid
--------------+----------------
          794 |          57542
(1 row)

UPDATE 1
locks=*#
```

##### Сессия №2

Обновляем значение 'amount' во второй строке:
```
locks=# BEGIN;
SELECT txid_current(), pg_backend_pid();
UPDATE accounts SET amount = amount - 200.00 WHERE acc_no = 2;
BEGIN
 txid_current | pg_backend_pid
--------------+----------------
          795 |          57548
(1 row)

UPDATE 1
locks=*#
```

##### Сессия №3

Обновляем значение 'amount' в третьей строке:
```
locks=# BEGIN;
SELECT txid_current(), pg_backend_pid();
UPDATE accounts SET amount = amount - 300.00 WHERE acc_no = 3;
BEGIN
 txid_current | pg_backend_pid
--------------+----------------
          796 |          57556
(1 row)

UPDATE 1
locks=*#
```

Далее непосредственно симулируем взаимоблокировку:

##### Сессия №1

Обновляем значение 'amount' во второй строке (при этом сеанс зависает на ожидании):
```
locks=*# UPDATE accounts SET amount = amount + 100.00 WHERE acc_no = 2;
```


##### Сессия №2

Обновляем значение 'amount' в третьей строке (значение успешно обновится):
```
locks=*# UPDATE accounts SET amount = amount + 100.00 WHERE acc_no = 3;
UPDATE 1
locks=*#
```

##### Сессия №3

Пытаемся обновить значение 'amount' в первой строке (возникает ошибка 'deadlock detected'):

```
locks=*# UPDATE accounts SET amount = amount + 100.00 WHERE acc_no = 1;
ERROR:  deadlock detected
DETAIL:  Process 57556 waits for ShareLock on transaction 794; blocked by process 57542.
Process 57542 waits for ShareLock on transaction 795; blocked by process 57548.
Process 57548 waits for ShareLock on transaction 796; blocked by process 57556.
HINT:  See server log for query details.
CONTEXT:  while updating tuple (0,39) in relation "accounts"
locks=!#
```

##### Итог:
Возникла взаимоблокировка, т. к. сессия №3 ждала завершения транзакций в сессиях №1 и №2, а сессия №2 ожидала завершения транзации в сессии №3

В целом, анализ записей лога поможет понять, какие транзакции блокировали друг друга и какие запросы вызвали блокировку:

```
2023-05-24 20:03:33.520 UTC [57556] postgres@locks ERROR:  deadlock detected
2023-05-24 20:03:33.520 UTC [57556] postgres@locks DETAIL:  Process 57556 waits for ShareLock on transaction 794; blocked by process 57542.
	Process 57542 waits for ShareLock on transaction 795; blocked by process 57548.
	Process 57548 waits for ShareLock on transaction 796; blocked by process 57556.
	Process 57556: UPDATE accounts SET amount = amount + 100.00 WHERE acc_no = 1;
	Process 57542: UPDATE accounts SET amount = amount + 100.00 WHERE acc_no = 2;
	Process 57548: UPDATE accounts SET amount = amount + 100.00 WHERE acc_no = 3;
2023-05-24 20:03:33.520 UTC [57556] postgres@locks HINT:  See server log for query details.
2023-05-24 20:03:33.520 UTC [57556] postgres@locks CONTEXT:  while updating tuple (0,39) in relation "accounts"
2023-05-24 20:03:33.520 UTC [57556] postgres@locks STATEMENT:  UPDATE accounts SET amount = amount + 100.00 WHERE acc_no = 1;
2023-05-24 20:03:33.520 UTC [57548] postgres@locks LOG:  process 57548 acquired ShareLock on transaction 796 after 969.490 ms
2023-05-24 20:03:33.520 UTC [57548] postgres@locks CONTEXT:  while updating tuple (0,41) in relation "accounts"
2023-05-24 20:03:33.520 UTC [57548] postgres@locks STATEMENT:  UPDATE accounts SET amount = amount + 100.00 WHERE acc_no = 3;

root@otus:~#
```
