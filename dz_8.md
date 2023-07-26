### Настройка PostgreSQL

Нагрузочное тестирование и тюнинг PostgreSQL

Цель:

- сделать нагрузочное тестирование PostgreSQL
- настроить параметры PostgreSQL для достижения максимальной производительности


###### ✅ развернуть виртуальную машину любым удобным способом

Для проекта выбрана облачная платформа Yandex Cloud;
	VM: OS Ubuntu 22.04 LTS/2 vCPU/2GB RAM/SSD 18 

###### ✅ поставить на неё PostgreSQL 15 любым способом

PostgreSQL установлен командой:

```
sudo apt update && sudo apt upgrade -y && sudo sh -c 'echo "deb http://apt.postgresql.org/pub/repos/apt $(lsb_release -cs)-pgdg main" > /etc/apt/sources.list.d/pgdg.list' && wget --quiet -O - https://www.postgresql.org/media/keys/ACCC4CF8.asc | sudo apt-key add - && sudo apt-get update && sudo apt-get -y install postgresql-15
```

```
root@otus:~# pg_config --version
PostgreSQL 15.3 (Ubuntu 15.3-1.pgdg22.04+1)

root@otus:~#
```

###### ✅ настроить кластер PostgreSQL 15 на максимальную производительность не обращая внимание на возможные проблемы с надежностью в случае аварийной перезагрузки виртуальной машины  

Будут рассматриваться несколько конфигураций


###### ✅ нагрузить кластер через утилиту через утилиту pgbench ([https://postgrespro.ru/docs/postgrespro/14/pgbench](https://postgrespro.ru/docs/postgrespro/14/pgbench "https://postgrespro.ru/docs/postgrespro/14/pgbench"))  

#### Тест №1. Дефолтная конфигурация

#### Итоговый tps = 406.261459

Результаты теста:

```
root@otus:/etc/postgresql/15/main/conf.d# sudo -u postgres pgbench -c 50 -j 2 -P 10 -T 30 otus
pgbench (15.3 (Ubuntu 15.3-1.pgdg22.04+1))
starting vacuum...end.
progress: 10.0 s, 444.3 tps, lat 111.033 ms stddev 118.166, 0 failed
progress: 20.0 s, 294.9 tps, lat 167.297 ms stddev 153.433, 0 failed
progress: 30.0 s, 477.9 tps, lat 105.676 ms stddev 107.287, 0 failed
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 50
number of threads: 2
maximum number of tries: 1
duration: 30 s
number of transactions actually processed: 12221
number of failed transactions: 0 (0.000%)
latency average = 122.818 ms
latency stddev = 126.226 ms
initial connection time = 59.010 ms
tps = 406.261459 (without initial connection time)

root@otus:
```


#### Тест №2. Конфигурация рассчитаная pgtune

#### Итоговый tps = 537.801832** (+35% к стандатной конфигурации)

Получаем подходящие параметры для сервера с помощью pgtune и закидываем их в include-файл в директорию  /etc/postgresql/15/main/conf.d и перезапускаем кластер.

Конфигурация предложенная сервисом pgtune:

```
# DB Version: 15
# OS Type: linux
# DB Type: web
# Total Memory (RAM): 2 GB
# CPUs num: 1
# Connections num: 20
# Data Storage: ssd

max_connections = 20
shared_buffers = 512MB
effective_cache_size = 1536MB
maintenance_work_mem = 128MB
checkpoint_completion_target = 0.9
wal_buffers = 16MB
default_statistics_target = 100
random_page_cost = 1.1
effective_io_concurrency = 200
work_mem = 13107kB
min_wal_size = 1GB
max_wal_size = 4GB
```

Результаты теста:

```
root@otus:/etc/postgresql/15/main/conf.d# sudo -u postgres pgbench -c 50 -j 2 -P 10 -T 30 otus
pgbench (15.3 (Ubuntu 15.3-1.pgdg22.04+1))
starting vacuum...end.
progress: 10.0 s, 558.8 tps, lat 87.873 ms stddev 66.917, 0 failed
progress: 20.0 s, 543.7 tps, lat 91.602 ms stddev 88.166, 0 failed
progress: 30.0 s, 508.8 tps, lat 99.326 ms stddev 111.449, 0 failed
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 50
number of threads: 2
maximum number of tries: 1
duration: 30 s
number of transactions actually processed: 16163
number of failed transactions: 0 (0.000%)
latency average = 92.827 ms
latency stddev = 90.046 ms
initial connection time = 44.390 ms
tps = 537.801832 (without initial connection time)

root@otus:
```

#### Тест №3. Поднимаем лимиты потребления RAM

#### Итоговый tps = 617.277447** (+55% к стандатной конфигурации)

Корректируем следующие параметры:

shared_buffers = 1024MB # увеличиваем объем памяти для кэширования данныхс 512 до 1024 MB;

wal_buffers = 64MB # увеличиваем с 16 до 64 MB чтобы обеспечить больший буфер записи журнала транзакций;

work_mem = 65536kB # увеличиваем с 13107kB до 65536kB чтобы позволить больше памяти для временных операций сортировки;


Результаты теста:

```
root@otus:/etc/postgresql/15/main/conf.d# sudo -u postgres pgbench -c 50 -j 2 -P 10 -T 30 otus
pgbench (15.3 (Ubuntu 15.3-1.pgdg22.04+1))
starting vacuum...end.
progress: 10.0 s, 706.3 tps, lat 70.032 ms stddev 55.743, 0 failed
progress: 20.0 s, 641.8 tps, lat 77.910 ms stddev 77.462, 0 failed
progress: 30.0 s, 504.4 tps, lat 99.054 ms stddev 100.885, 0 failed
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 50
number of threads: 2
maximum number of tries: 1
duration: 30 s
number of transactions actually processed: 18573
number of failed transactions: 0 (0.000%)
latency average = 80.817 ms
latency stddev = 78.554 ms
initial connection time = 58.458 ms
tps = 617.277447 (without initial connection time)

root@otus:
```


#### Тест №4. Вишенка на торте

#### Итоговый tps = 2798.009737** (+600% к стандатной конфигурации)

Так как основным условием данного тестирования является получение максимальной производительности и мы готовы пожертвовать надежностьютью, то обращаем внимание на выбор значения для параметра 'synchronous_commit', который отвечает за режим синхронизации записи транзакций на диск.

Добавляем в include-файл параметр synchronous_commit = off и перезапускаем кластер.

Таким образом, PostgreSQL будет считать транзакцию завершенной сразу после записи данных в кэш и не будет дожидается подтверждения о физической записи на диск, что даст значительный прирост производительности.

Результаты теста:

```
root@otus:/etc/postgresql/15/main/conf.d# sudo -u postgres pgbench -c 50 -j 2 -P 10 -T 30 otus
pgbench (15.3 (Ubuntu 15.3-1.pgdg22.04+1))
starting vacuum...end.
progress: 10.0 s, 2768.7 tps, lat 17.921 ms stddev 11.753, 0 failed
progress: 20.0 s, 2866.5 tps, lat 17.451 ms stddev 14.612, 0 failed
progress: 30.0 s, 2760.3 tps, lat 18.114 ms stddev 12.764, 0 failed
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 50
number of threads: 2
maximum number of tries: 1
duration: 30 s
number of transactions actually processed: 84004
number of failed transactions: 0 (0.000%)
latency average = 17.845 ms
latency stddev = 13.152 ms
initial connection time = 58.671 ms
tps = 2798.009737 (without initial connection time)

root@otus:
```
