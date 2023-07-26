## Работа с журналами

Цель:

- уметь работать с журналами и контрольными точками
- уметь настраивать параметры журналов
  


###### ✅ 1. Настройте выполнение контрольной точки раз в 30 секунд.

Меняем значение 'checkpoint_timeout' в файле postgresql.conf и перезапускаем службу PostgreSQL.
Дополнительно делаем запрос к pg_settings чтобы убедиться, что значение параметра изменилось:

```
postgres=# SELECT name, setting FROM pg_settings WHERE name = 'checkpoint_timeout';

        name        | setting 

--------------------+---------

 checkpoint_timeout | 30

(1 row)

postgres=#
```

###### ✅ 2. 10 минут c помощью утилиты pgbench подавайте нагрузку.

Смотрим размер директории WAL-файлов до теста (17MB):

```
root@otus:~# du -h /var/lib/postgresql/15/main/pg_wal/
4.0K /var/lib/postgresql/15/main/pg_wal/archive_status
17M /var/lib/postgresql/15/main/pg_wal/

root@otus:~#
```

Сбрасываем накопленную статистику bgwriter:

```
root@otus:~# sudo -u postgres psql -c "select pg_stat_reset_shared('bgwriter')"

 pg_stat_reset_shared 

----------------------

(1 row)

root@otus:~#
```


Запускаем pgbench:

```
root@otus:~# sudo -u postgres pgbench -c10 -P 20 -T 600 -U postgres otus
pgbench (15.3 (Ubuntu 15.3-1.pgdg22.04+1))
starting vacuum...end.
progress: 20.0 s, 486.7 tps, lat 20.516 ms stddev 21.238, 0 failed
progress: 40.0 s, 493.7 tps, lat 20.253 ms stddev 19.698, 0 failed
progress: 60.0 s, 644.2 tps, lat 15.517 ms stddev 15.486, 0 failed
progress: 80.0 s, 367.1 tps, lat 27.212 ms stddev 27.170, 0 failed
progress: 100.0 s, 454.3 tps, lat 22.037 ms stddev 19.301, 0 failed
progress: 120.0 s, 664.4 tps, lat 15.045 ms stddev 14.930, 0 failed
progress: 140.0 s, 451.8 tps, lat 22.144 ms stddev 22.388, 0 failed
progress: 160.0 s, 649.2 tps, lat 15.394 ms stddev 14.213, 0 failed
progress: 180.0 s, 351.4 tps, lat 28.469 ms stddev 30.212, 0 failed
progress: 200.0 s, 567.6 tps, lat 17.611 ms stddev 18.622, 0 failed
progress: 220.0 s, 471.9 tps, lat 21.191 ms stddev 19.702, 0 failed
progress: 240.0 s, 657.8 tps, lat 15.209 ms stddev 14.792, 0 failed
progress: 260.0 s, 521.4 tps, lat 19.179 ms stddev 24.793, 0 failed
progress: 280.0 s, 631.7 tps, lat 15.831 ms stddev 14.853, 0 failed
progress: 300.0 s, 689.7 tps, lat 14.480 ms stddev 13.194, 0 failed
progress: 320.0 s, 426.7 tps, lat 23.447 ms stddev 23.901, 0 failed
progress: 340.0 s, 617.0 tps, lat 16.217 ms stddev 19.809, 0 failed
progress: 360.0 s, 578.5 tps, lat 17.267 ms stddev 20.694, 0 failed
progress: 380.0 s, 410.1 tps, lat 24.407 ms stddev 26.675, 0 failed
progress: 400.0 s, 527.6 tps, lat 18.945 ms stddev 21.030, 0 failed
progress: 420.0 s, 617.3 tps, lat 16.209 ms stddev 18.368, 0 failed
progress: 440.0 s, 561.7 tps, lat 17.799 ms stddev 22.371, 0 failed
progress: 460.0 s, 688.7 tps, lat 14.522 ms stddev 11.326, 0 failed
progress: 480.0 s, 483.1 tps, lat 20.699 ms stddev 25.273, 0 failed
progress: 500.0 s, 461.2 tps, lat 21.636 ms stddev 18.163, 0 failed
progress: 520.0 s, 447.2 tps, lat 22.409 ms stddev 21.323, 0 failed
progress: 540.0 s, 578.9 tps, lat 17.274 ms stddev 18.182, 0 failed
progress: 560.0 s, 486.6 tps, lat 20.550 ms stddev 19.096, 0 failed
progress: 580.0 s, 717.6 tps, lat 13.932 ms stddev 11.394, 0 failed
progress: 600.0 s, 602.2 tps, lat 16.608 ms stddev 17.528, 0 failed
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 10
number of threads: 1
maximum number of tries: 1
duration: 600 s
number of transactions actually processed: 326155
number of failed transactions: 0 (0.000%)
latency average = 18.395 ms
latency stddev = 19.632 ms
initial connection time = 21.214 ms
tps = 543.590099 (without initial connection time)
```


После завершения теста смотрим статистику (по расписанию выполнилось 22 контрольные точки ('checkpoints_timed')):

```
root@otus:~# sudo -u postgres psql
could not change directory to "/root": Permission denied
psql (15.3 (Ubuntu 15.3-1.pgdg22.04+1))
Type "help" for help.

postgres=# SELECT * FROM pg_stat_bgwriter \gx
-[ RECORD 1 ]---------+------------------------------
checkpoints_timed     | 22
checkpoints_req       | 0
checkpoint_write_time | 537755
checkpoint_sync_time  | 273
buffers_checkpoint    | 40151
buffers_clean         | 0
maxwritten_clean      | 0
buffers_backend       | 2271
buffers_backend_fsync | 0
buffers_alloc         | 2545
stats_reset           | 2023-07-26 16:38:31.228246+00

postgres=#
```



###### ✅ 3. Измерьте, какой объем журнальных файлов был сгенерирован за это время. Оцените, какой объем приходится в среднем на одну контрольную точку.

Если проверить размер директории WAL-файлов после теста, то увидим увеличение до 65МБ (+48МБ):

```
root@otus:~# du -h /var/lib/postgresql/15/main/pg_wal/

4.0K	/var/lib/postgresql/15/main/pg_wal/archive_status
65M	/var/lib/postgresql/15/main/pg_wal/

```

Но, если обратиться к логу PostgreSQL, то заметим ключевые моменты:

1. 20 созданных контрольных точек;
2. среднее полное время создания контрольной точки 26.8 сек;
3. среднее значение параметра 'estimate' (объем сбрасываемых данных из буферного кэша на диск, а также фактически "расстояние" между чекпоинтами) примерно 22МБ.

Исходя из этого делаем вывод - общий объем данных записываемых в WAL составил около 450 МБ, неактуальные данные очищались - поэтому визуально мы видим рост размера директории WAL только до 65МБ.


Ниже небольшой кусок лога:

```
2023-07-26 16:39:01.370 UTC [27862] LOG:  checkpoint starting: time

2023-07-26 16:39:28.021 UTC [27862] LOG:  checkpoint complete: wrote 1989 buffers (12.1%); 0 WAL file(s) added, 0 removed, 1 recycled; write=26.584 s, sync=0.012 s, total=26.651 s; sync files=24, longest=0.006 s, average=0.001 s; distance=17942 kB, estimate=17942 kB

2023-07-26 16:39:31.024 UTC [27862] LOG:  checkpoint starting: time

2023-07-26 16:39:58.153 UTC [27862] LOG:  checkpoint complete: wrote 1891 buffers (11.5%); 0 WAL file(s) added, 0 removed, 1 recycled; write=27.000 s, sync=0.033 s, total=27.130 s; sync files=12, longest=0.015 s, average=0.003 s; distance=21029 kB, estimate=21029 kB

2023-07-26 16:40:01.156 UTC [27862] LOG:  checkpoint starting: time

2023-07-26 16:40:28.035 UTC [27862] LOG:  checkpoint complete: wrote 1988 buffers (12.1%); 0 WAL file(s) added, 0 removed, 2 recycled; write=26.809 s, sync=0.007 s, total=26.879 s; sync files=17, longest=0.004 s, average=0.001 s; distance=21958 kB, estimate=21958 kB
...
```


###### ✅ 4. Проверьте данные статистики: все ли контрольные точки выполнялись точно по расписанию. Почему так произошло?

В данном случае pgbench запускался со скромными параметрами (10 клиентов, 20 транзаций), поэтому чекпоинты создавались вовремя и следовали друг за другом каждые 30 секунд.

Если дать более серьезную нагрузку, то затрачиваемое время записи данных на диск, а в конечном счете и фактическое время создания контрольной точки увеличится и соответственно, создание следующей (очереденой) контрольной точки начнется только после создания предыдущей, игнорируя заданный период (30 сек).

###### ✅ 5. Сравните tps в синхронном/асинхронном режиме утилитой pgbench. Объясните полученный результат.


Запускаем тест в синхронном режиме (tps = 391.623947):

```
root@otus:~# sudo -u postgres pgbench -c10 -P 50 -U postgres otus
pgbench (15.3 (Ubuntu 15.3-1.pgdg22.04+1))
starting vacuum...end.
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 10
number of threads: 1
maximum number of tries: 1
number of transactions per client: 10
number of transactions actually processed: 100/100
number of failed transactions: 0 (0.000%)
latency average = 17.970 ms
latency stddev = 19.734 ms
initial connection time = 23.005 ms
tps = 391.623947 (without initial connection time)

root@otus:~#
```


Устанавливаем значение 'synchronous_commit' на 'off' и перезапускаем службу PostgreSQL:

```
root@otus:~# echo "synchronous_commit = off" | sudo tee -a /etc/postgresql/15/main/postgresql.conf
synchronous_commit = off

root@otus:~# systemctl restart postgresql
```

Запускаем тест ув асинхронном режиме (tps = 2661.769012):

```
root@otus:~# sudo -u postgres pgbench -c10 -P 50 -U postgres otus
pgbench (15.3 (Ubuntu 15.3-1.pgdg22.04+1))
starting vacuum...end.
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 10
number of threads: 1
maximum number of tries: 1
number of transactions per client: 10
number of transactions actually processed: 100/100
number of failed transactions: 0 (0.000%)
latency average = 2.983 ms
latency stddev = 3.248 ms
initial connection time = 20.517 ms
tps = 2661.769012 (without initial connection time)

root@otus:~#
```

Как итог разница в производительности в 6 раз.

В асинхронном режиме транзакция будет считаться завершенной как только изменения будут записаны в буферный кэш (без ожидания дополнительного подтверждение записи на диск), что в итоге повышает производительность, но и порождает риск потери данных в случае сбоя.

###### ✅ 6. Создайте новый кластер с включенной контрольной суммой страниц. Создайте таблицу. Вставьте несколько значений. Выключите кластер. Измените пару байт в таблице. Включите кластер и сделайте выборку из таблицы. Что и почему произошло? как проигнорировать ошибку и продолжить работу?


Создаем и запускаем новый кластер с включеной проверкой контрольных сумм:

```
root@otus:~# sudo pg_createcluster 15 otus -p 5433 -- --data-checksums
Creating new PostgreSQL cluster 15/otus ...
/usr/lib/postgresql/15/bin/initdb -D /var/lib/postgresql/15/otus --auth-local peer --auth-host scram-sha-256 --no-instructions --data-checksums
The files belonging to this database system will be owned by user "postgres".
This user must also own the server process.

The database cluster will be initialized with locale "en_US.UTF-8".
The default database encoding has accordingly been set to "UTF8".
The default text search configuration will be set to "english".

Data page checksums are enabled.

fixing permissions on existing directory /var/lib/postgresql/15/otus ... ok
creating subdirectories ... ok
selecting dynamic shared memory implementation ... posix
selecting default max_connections ... 100
selecting default shared_buffers ... 128MB
selecting default time zone ... Etc/UTC
creating configuration files ... ok
running bootstrap script ... ok
performing post-bootstrap initialization ... ok
syncing data to disk ... ok
Ver Cluster Port Status Owner    Data directory              Log file
15  otus    5433 down   postgres /var/lib/postgresql/15/otus /var/log/postgresql/postgresql-15-otus.log

root@otus:~# sudo pg_ctlcluster 15 otus start
```

Создаем таблицу и вставляем значения:

```
root@otus:~# sudo -u postgres psql -p 5433 -c "CREATE TABLE journal (number integer)"

CREATE TABLE

root@otus:~# sudo -u postgres psql -p 5433 -c "INSERT INTO journal (number) SELECT generate_series(1, 10)"

INSERT 0 10

root@otus:~#
```

Делаем запрос для получения пути к нашей таблице в файловой системе:

```

root@otus:~# sudo -u postgres psql -p 5433 -c "SELECT pg_relation_filepath('journal');"

 pg_relation_filepath 

----------------------

 base/5/16394

(1 row)
```

Останавливаем кластер и меняем несколько байтов в странице (сотрем из заголовка LSN последней журнальной записи)

```
root@otus:~# sudo pg_ctlcluster 15 otus stop
root@otus:~# dd if=/dev/zero of=/var/lib/postgresql/15/otus/base/5/16394 oflag=dsync conv=notrunc bs=1 count=8

8+0 records in
8+0 records out
8 bytes copied, 0.0404818 s, 0.2 kB/s

root@otus:~#
```

Стартуем кластер и пытаемся посмотреть значения в таблице:

```
root@otus:~# sudo pg_ctlcluster 15 otus start

root@otus:~# sudo -u postgres psql -p 5433 -c "select * from journal"

could not change directory to "/root": Permission denied
WARNING:  page verification failed, calculated checksum 30361 but expected 20676

ERROR:  invalid page in block 0 of relation base/5/16394

root@otus:~#
```


Результат: при попытке прочитать данные из таблицы 'journal' PostgreSQL выдал ошибку и предупреждает, что данные в блоке с номером 0 содержат неверные значения контрольной суммы, т.е. одна из страниц таблицы повреждена или содержит неверные данные.

Варианты решения этой проблемы:

1. восстановиться из бэкапа;
2. указать PostgreSQL игнорировать ошибки (параметр 'zero_damaged_pages');
3. удалить поврежденные строки.
