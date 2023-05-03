## Установка и настройка PostgteSQL в контейнере Docker

Цель:

-   установить PostgreSQL в Docker контейнере
-   настроить контейнер для внешнего подключения


### Описание/Пошаговая инструкция выполнения домашнего задания:


###### ✅ Создать ВМ с Ubuntu 20.04/22.04 или развернуть докер любым удобным способом

> **Комментарий:** Создана VM на платформе Yandex Cloud;


###### ✅ Поставить на нем Docker Engine

> **Решение:** 

	Устанавливаем Docker Engine:

```
otus@otus:~$ curl -fsSL https://get.docker.com -o get-docker.sh && sudo sh get-docker.sh && rm get-docker.sh && sudo usermod -aG docker $USER
```

	 И проверяем версию:
	 
```
root@otus:/home/otus# docker version

Client: Docker Engine - Community
 Version:           23.0.5
 API version:       1.42
 Go version:        go1.19.8
 Git commit:        bc4487a
 Built:             Wed Apr 26 16:21:07 2023
 OS/Arch:           linux/amd64
 Context:           default

Server: Docker Engine - Community
 Engine:
  Version:          23.0.5
  API version:      1.42 (minimum version 1.12)
  Go version:       go1.19.8
  Git commit:       94d3ad6
  Built:            Wed Apr 26 16:21:07 2023
  OS/Arch:          linux/amd64
  Experimental:     false
 containerd:
  Version:          1.6.20
  GitCommit:        2806fc1057397dbaeefbea0e4e17bddfbd388f38
 runc:
  Version:          1.1.5
  GitCommit:        v1.1.5-0-gf19387a
 docker-init:
  Version:          0.19.0
  GitCommit:        de40ad0
```


###### ✅ Сделать каталог /var/lib/postgres

> **Решение:** 

	Создаем каталог командой 'mkdir':

```
root@otus:/home/otus# mkdir /var/lib/postgres
```


###### ✅  Развернуть контейнер с PostgreSQL 15 смонтировав в него /var/lib/postgresql 


>**Решение:**

	Cоздаем docker-сеть:
	
```
root@otus:/home/otus# docker network create pg-net

35916ee2ef6456bc778c5767fc5bfc62fd691501859d0f0f0dad07fe67088995
```

	И запускаем контейнер с сервером Postgres:
	
```
otus@otus:~$ docker run --name pg-server --network pg-net -e POSTGRES_PASSWORD=postgres -d -p 5432:5432 -v /var/lib/postgres:/var/lib/postgresql/data postgres:15

60d1817d1369c1d2668d6c54b2bcbf62c767dbbe21a550f47b741788a56ed89b

```


###### ✅ Развернуть контейнер с клиентом postgres. Подключится из контейнера с клиентом к контейнеру с сервером и сделать таблицу с парой строк

>**Решение:** 

	Запускаем контейнер с клиентом Postgres c инициализацией подключения к серверу Postgres:
	
```
otus@otus:~$ docker run -it --rm --network pg-net --name pg-client postgres:15 psql -h pg-server -U postgres
Password for user postgres:
psql (15.2 (Debian 15.2-1.pgdg110+1))
Type "help" for help.
```

	Смотрим список всех БД:
	
```
postgres=# \l
                                                List of databases
   Name    |  Owner   | Encoding |  Collate   |   Ctype    | ICU Locale | Locale Provider |   Access privileges
-----------+----------+----------+------------+------------+------------+-----------------+-----------------------
 postgres  | postgres | UTF8     | en_US.utf8 | en_US.utf8 |            | libc            |
 template0 | postgres | UTF8     | en_US.utf8 | en_US.utf8 |            | libc            | =c/postgres          +
           |          |          |            |            |            |                 | postgres=CTc/postgres
 template1 | postgres | UTF8     | en_US.utf8 | en_US.utf8 |            | libc            | =c/postgres          +
           |          |          |            |            |            |                 | postgres=CTc/postgres
(3 rows)
```

	Создаем базу 'otus':
	
```
postgres=# CREATE DATABASE otus;
CREATE DATABASE
postgres=# CREATE TABLE test (
  id SERIAL PRIMARY KEY,
  name VARCHAR(50) NOT NULL,
  email VARCHAR(50) NOT NULL
);
CREATE TABLE
```

	Убеждаемся что созданная БД 'otus' отображается в списке баз:
	
```
postgres=# \l
                                                List of databases
   Name    |  Owner   | Encoding |  Collate   |   Ctype    | ICU Locale | Locale Provider |   Access privileges
-----------+----------+----------+------------+------------+------------+-----------------+-----------------------
 otus      | postgres | UTF8     | en_US.utf8 | en_US.utf8 |            | libc            |
 postgres  | postgres | UTF8     | en_US.utf8 | en_US.utf8 |            | libc            |
 template0 | postgres | UTF8     | en_US.utf8 | en_US.utf8 |            | libc            | =c/postgres          +
           |          |          |            |            |            |                 | postgres=CTc/postgres
 template1 | postgres | UTF8     | en_US.utf8 | en_US.utf8 |            | libc            | =c/postgres          +
           |          |          |            |            |            |                 | postgres=CTc/postgres
(4 rows)

postgres=#

```

###### ✅ Подключиться к контейнеру с сервером с ноутбука/компьютера извне инстансов GCP/ЯО/места установки докера 

>**Решение:** 

	Подключаемся к VM по SSH:

```
~/ ssh otus@158.160.32.204
Welcome to Ubuntu 22.04.2 LTS (GNU/Linux 5.15.0-71-generic x86_64)
```

	Проверяем список всех контейнеров:
```
otus@otus:~$ docker ps -a
CONTAINER ID   IMAGE         COMMAND                  CREATED          STATUS          PORTS                                       NAMES
60d1817d1369   postgres:15   "docker-entrypoint.s…"   3 minutes ago   Up 3 minutes   0.0.0.0:5432->5432/tcp, :::5432->5432/tcp   pg-server
otus@otus:~$

```

###### ✅ Удалить контейнер с сервером

>**Решение:** 

	Смотрим ID контейнера с сервером Postgres:

```
otus@otus:~$ docker ps -a
CONTAINER ID   IMAGE         COMMAND                  CREATED          STATUS         PORTS                                       NAMES
60d1817d1369   postgres:15   "docker-entrypoint.s…"   5 minutes ago   Up 5 seconds   0.0.0.0:5432->5432/tcp, :::5432->5432/tcp   pg-server
```

	Удаляем контейнер используя флаг '--force'
```
otus@otus:~$ docker rm -f 60d1817d1369
60d1817d1369

otus@otus:~$
```

###### ✅ Создать его заново 

>**Решение:** 

	Запускаем контейнер с сервером Postgres:
	
```
otus@otus:~$ docker run --name pg-server --network pg-net -e POSTGRES_PASSWORD=postgres -d -p 5432:5432 -v /var/lib/postgres:/var/lib/postgresql/data postgres:15

8b120763a8e33b80cbd57d65cf1d1481e4a6e19591c0932c66539bc0421e241d
```

	Проверяем список всех контейнеров:
	
```
otus@otus:~$ docker ps -a
CONTAINER ID   IMAGE         COMMAND                  CREATED         STATUS         PORTS                                       NAMES
8b120763a8e3   postgres:15   "docker-entrypoint.s…"   9 seconds ago   Up 7 seconds   0.0.0.0:5432->5432/tcp, :::5432->5432/tcp   pg-server

otus@otus:~$
```

###### ✅  Подключится снова из контейнера с клиентом к контейнеру с сервером. Проверить, что данные остались на месте 

>**Решение:** 

	Инициализируем подключение к серверу Postgres через контейнер с клиентом:
```
otus@otus:~$ docker run -it --rm --network pg-net --name pg-client postgres:15 psql -h pg-server -U postgres
Password for user postgres:
psql (15.2 (Debian 15.2-1.pgdg110+1))
Type "help" for help.
```

	Смотрим список всех БД, 'otus' на месте:
	
```
postgres=# \l
                                                List of databases
   Name    |  Owner   | Encoding |  Collate   |   Ctype    | ICU Locale | Locale Provider |   Access privileges
-----------+----------+----------+------------+------------+------------+-----------------+-----------------------
 otus      | postgres | UTF8     | en_US.utf8 | en_US.utf8 |            | libc            |
 postgres  | postgres | UTF8     | en_US.utf8 | en_US.utf8 |            | libc            |
 template0 | postgres | UTF8     | en_US.utf8 | en_US.utf8 |            | libc            | =c/postgres          +
           |          |          |            |            |            |                 | postgres=CTc/postgres
 template1 | postgres | UTF8     | en_US.utf8 | en_US.utf8 |            | libc            | =c/postgres          +
           |          |          |            |            |            |                 | postgres=CTc/postgres
(4 rows)

postgres=#
```
