## Установка и настройка PostgreSQL

Цель:

-   создавать дополнительный диск для уже существующей виртуальной машины, размечать его и делать на нем файловую систему
-   переносить содержимое базы данных PostgreSQL на дополнительный диск
-   переносить содержимое БД PostgreSQL между виртуальными машинами

  

### Описание/Пошаговая инструкция выполнения домашнего задания:

###### ✅ Создайте виртуальную машину c Ubuntu 20.04/22.04 LTS в GCE/ЯО/Virtual Box/докере

	Для проекта выбрана облачная платформа Yandex Cloud;
	VM: OS Ubuntu 22.04 LTS/2 vCPU/2GB RAM/SSD 18 


###### ✅ Поставьте на нее PostgreSQL 15 через sudo apt

Выполено:
```
root@otus:~# pg_config --version
PostgreSQL 15.3 (Ubuntu 15.3-1.pgdg22.04+1)

root@otus:~#
```


###### ✅ Проверьте что кластер запущен через sudo -u postgres pg_lsclusters

Кластер запущен:
```
root@otus:~# sudo -u postgres pg_lsclusters
Ver Cluster Port Status Owner    Data directory              Log file
15  main    5432 online postgres /var/lib/postgresql/15/main /var/log/postgresql/postgresql-15-main.log

root@otus:~#
```

###### ✅ Зайдите из под пользователя postgres в psql и сделайте произвольную таблицу с произвольным содержимым 

Выполнено:
```
root@otus:/etc/postgresql/15/main# sudo -u postgres psql
psql (15.3 (Ubuntu 15.3-1.pgdg22.04+1))
Type "help" for help.

postgres=# create table test(c1 text);
CREATE TABLE

postgres=# insert into test values('1');
INSERT 0 1

postgres=# \q
root@otus:/etc/postgresql/15/main#
```

###### ✅ Остановите postgres например через sudo -u postgres pg_ctlcluster 15 main stop

Выполнено:
```
root@otus:/etc/postgresql/15/main# systemctl stop postgresql
root@otus:/etc/postgresql/15/main# systemctl status postgresql
● postgresql.service - PostgreSQL RDBMS
     Loaded: loaded (/lib/systemd/system/postgresql.service; enabled; vendor preset: enabled)
     Active: inactive (dead) since Mon 2023-05-22 16:34:37 UTC; 1s ago
   Main PID: 48335 (code=exited, status=0/SUCCESS)
        CPU: 1ms

May 22 16:14:45 otus systemd[1]: Starting PostgreSQL RDBMS...
May 22 16:14:45 otus systemd[1]: Finished PostgreSQL RDBMS.
May 22 16:34:37 otus systemd[1]: postgresql.service: Deactivated successfully.
May 22 16:34:37 otus systemd[1]: Stopped PostgreSQL RDBMS.

root@otus:/etc/postgresql/15/main#
```

###### ✅ Создайте новый диск к ВМ размером 10GB

Выполнено. В Консоли управления дисками Yandex Cloud был создан дполнительный диск;


###### ✅ Добавьте свеже-созданный диск к виртуальной машине - надо зайти в режим ее редактирования и дальше выбрать пункт attach existing disk

Выполнено - диск подключен к созданной VM;


###### ✅ Проинициализируйте диск согласно инструкции и подмонтировать файловую систему, только не забывайте менять имя диска на актуальное, в вашем случае это скорее всего будет /dev/sdb - [https://www.digitalocean.com/community/tutorials/how-to-partition-and-format-storage-devices-in-linux](https://www.digitalocean.com/community/tutorials/how-to-partition-and-format-storage-devices-in-linux "https://www.digitalocean.com/community/tutorials/how-to-partition-and-format-storage-devices-in-linux")


Добавленный диск (vdb) был проинициализирован, создан раздел с файловой системой Ext4:

```
root@otus:~# sudo lsblk --fs
NAME   FSTYPE   FSVER LABEL         UUID                                 FSAVAIL FSUSE% MOUNTPOINTS
loop0  squashfs 4.0                                                            0   100% /snap/core20/1879
loop1  squashfs 4.0                                                            0   100% /snap/core20/1891
loop2  squashfs 4.0                                                            0   100% /snap/lxd/22923
loop3  squashfs 4.0                                                            0   100% /snap/lxd/24322
loop4  squashfs 4.0                                                            0   100% /snap/snapd/19122
vda
├─vda1
└─vda2 ext4     1.0                 82aeea96-6d42-49e6-85d5-9071d3c9b6aa   12.3G    26% /
vdb    ext4     1.0   datapartition f86a0470-0c5f-4f87-b251-3e4b544be5f5    9.2G     0% /mnt/data

root@otus:~#
```

###### ✅ Перезагрузите инстанс и убедитесь, что диск остается примонтированным (если не так смотрим в сторону fstab)

В fstab прописан параметр автомонтирования диска:
```
/dev/disk/by-uuid/f86a0470-0c5f-4f87-b251-3e4b544be5f5 /mnt/data ext4 defaults 0 2
```

После перезагрузки диск был автоматически примонтирован:
```
root@otus:~# df -h -x tmpfs
Filesystem      Size  Used Avail Use% Mounted on
/dev/vda2        18G  4.6G   13G  27% /
/dev/vdb        9.8G   24K  9.3G   1% /mnt/data

root@otus:~#
```


###### ✅ Сделайте пользователя postgres владельцем /mnt/data - chown -R postgres:postgres /mnt/data/

Выполнено:
```
root@otus:~# ls -ll /mnt
total 4
drwxr-xr-x 3 postgres postgres 4096 May 22 16:42 data
root@otus:~#
```


###### ✅ Перенесите содержимое /var/lib/postgres/15 в /mnt/data - mv /var/lib/postgresql/15/mnt/data

Выполнено:
```
root@otus:~# mv /var/lib/postgresql/15 /mnt/data/
root@otus:~# cd /mnt/data/15/main/

root@otus:/mnt/data/15/main# ls
base          pg_dynshmem   pg_notify    pg_snapshots  pg_subtrans  PG_VERSION  postgresql.auto.conf
global        pg_logical    pg_replslot  pg_stat       pg_tblspc    pg_wal      postmaster.opts
pg_commit_ts  pg_multixact  pg_serial    pg_stat_tmp   pg_twophase  pg_xact     postmaster.pid

root@otus:/mnt/data/15/main#
```


###### ✅ Попытайтесь запустить кластер - sudo -u postgres pg_ctlcluster 15 main start. Напишите получилось или нет и почему

Служба PostgreSQL не сможет запуститься должным образом, т.к. не будут обнаружены файлы базы данных в исходном каталоге '/var/lib/postgresql/15/', поскольку они были перемещены в '/mnt/data/'.




###### ✅ Задание: найти конфигурационный параметр в файлах раположенных в /etc/postgresql/15/main который надо поменять и поменяйте его. Напишите что и почему поменяли

Необходимо изменить параметр 'data_directory' (каталог размещения файлов баз данных и рабочих компонентов кластера Postgres) в конфигурационном файле postgresql.conf, чтобы он указывал на новый каталог:

data_directory = '/mnt/data/15/main/'


###### ✅ Попытайтесь запустить кластер - sudo -u postgres pg_ctlcluster 15 main start. Напишите получилось или нет и почему

Служба успешно запустилась, каталог для хранения данных определяется по новому заданному пути '/mnt/data/15/main':

```
root@otus:~# systemctl restart postgresql
root@otus:~# systemctl status postgresql
● postgresql.service - PostgreSQL RDBMS
     Loaded: loaded (/lib/systemd/system/postgresql.service; enabled; vendor preset: enabled)
     Active: active (exited) since Mon 2023-05-22 17:49:39 UTC; 6s ago
    Process: 1546 ExecStart=/bin/true (code=exited, status=0/SUCCESS)
   Main PID: 1546 (code=exited, status=0/SUCCESS)
        CPU: 915us

May 22 17:49:39 otus systemd[1]: Starting PostgreSQL RDBMS...
May 22 17:49:39 otus systemd[1]: Finished PostgreSQL RDBMS.
root@otus:~# sudo -u postgres psql
could not change directory to "/root": Permission denied
psql (15.3 (Ubuntu 15.3-1.pgdg22.04+1))
Type "help" for help.

postgres=# show data_directory;
  data_directory
-------------------
 /mnt/data/15/main
(1 row)

postgres=#
```


###### ✅ Зайдите через через psql и проверьте содержимое ранее созданной таблицы

Выполнено. Таблица и ее содержимое на месте:

```
postgres=# \d+
                                  List of relations
 Schema | Name | Type  |  Owner   | Persistence | Access method | Size  | Description
--------+------+-------+----------+-------------+---------------+-------+-------------
 public | test | table | postgres | permanent   | heap          | 16 kB |
(1 row)

postgres=# \d test
              Table "public.test"
 Column | Type | Collation | Nullable | Default
--------+------+-----------+----------+---------
 c1     | text |           |          |

postgres=#
```


###### ✅ Задание со звездочкой *: не удаляя существующий инстанс ВМ сделайте новый, поставьте на его PostgreSQL, удалите файлы с данными из /var/lib/postgres, перемонтируйте внешний диск который сделали ранее от первой виртуальной машины ко второй и запустите PostgreSQL на второй машине так чтобы он работал с данными на внешнем диске, расскажите как вы это сделали и что в итоге получилось.

1. На первом инстансе (VM_1) была остановлена служба Postgres и отмонтирован ранее добавленный диск;
2. В консоли управления Yandex Cloud в панели управления дисками указанный диск был отсоединен от VM_1;
3. Развернут аналогичный инстанс (VM_2) с PostgreSQL 15;
4. В консоли управления Yandex Cloud диск с данными был подключен к VM_2;
5. На VM_2 добавленный диск был определен в системе и примонтирован в заранее созданную директорию '/mnt/data';
6. Служба Postgres на VM_2 была остановлена, в конфигурационном файле postgresql.conf, изменен путь для параметра каталога данных (data_directory = '/mnt/data/15/main/');
7. Далее служба Postgres на VM_2 была запущена, осуществлено подключение через клиент psql и успешно проверено наличие таблицы 'test'.
