# Домашнее задание №4

## Работа с базами данных, пользователями и правами

### Данные о сервере

Для работы используется виртуальная машина на частном облаке со следующими характеристиками:

* CPU: 2 core Xeon E5-2690v2
* RAM: 4 Gb
* SSD: 20 Gb SSD
* ОС: Ubuntu 24.04 LTS

### Обновление ОС и установка необходимого ПО

Запускаем полное обновление ОС перед началом работы:

```
sudo apt-get update && sudo apt-get dist-upgrade -y
```
После окончания обновления при необходимости перезагружаем ВМ:

```
sudo reboot
```

Подключаем репозиторий PostgreSQL:

```
sudo apt install -y postgresql-common
sudo /usr/share/postgresql-common/pgdg/apt.postgresql.org.sh
```

Устанавливаем PostgreSQL 14:

```
sudo apt-get install postgresql-14 -y
```

Проверяем, что всё прошло успешно и запущено:

```
rsys@template:~$ pg_lsclusters
Ver Cluster Port Status Owner    Data directory              Log file
14  main    5432 online postgres /var/lib/postgresql/14/main /var/log/postgresql/postgresql-14-main.log
```

Подключаемся к postgresql с помощью psql, создаем базу, подключаемся к ней, создаем в базе схему, затем создаем таблицу и вставляем в нее строку:

```
rsys@template:~$ sudo -u postgres psql
psql (14.13 (Ubuntu 14.13-1.pgdg24.04+1))
Type "help" for help.

postgres=# CREATE DATABASE testdb;
CREATE DATABASE
postgres=# \c testdb
You are now connected to database "testdb" as user "postgres".
testdb=# CREATE SCHEMA testnm;
CREATE SCHEMA
testdb=# CREATE TABLE t1 (c1 integer);
CREATE TABLE
testdb=# INSERT INTO t1(c1) VALUES(1);
INSERT 0 1
```

Создаем новую роль readonly, даем ей права на подключение к базе testdb, на использование схемы testnm, на select всех таблиц схемы testnm:

```
testdb=# CREATE ROLE readonly;
CREATE ROLE
testdb=# GRANT CONNECT ON DATABASE testdb TO readonly;
GRANT
testdb=# GRANT USAGE ON SCHEMA testnm TO readonly;
GRANT
testdb=# GRANT SELECT ON ALL TABLES IN SCHEMA testnm TO readonly;
GRANT
```

Создаем пользователя testread, задаем ему пароль и даем роль readonly:

```
testdb=# CREATE USER testread WITH PASSWORD 'test123';
CREATE ROLE
testdb=# GRANT readonly TO testread;
GRANT ROLE
```

Подключаемся к базе testdb под пользователем testread:

```
rsys@template:~$ psql -h localhost -U testread -d testdb -W
Пароль:
psql (14.13 (Ubuntu 14.13-1.pgdg24.04+1))
SSL-соединение (протокол: TLSv1.3, шифр: TLS_AES_256_GCM_SHA384, бит: 256, сжатие: выкл.)
Введите "help", чтобы получить справку.

testdb=>
```

Выполняем запрос и получаем ошибку:

```
testdb=> SELECT * FROM t1;
ОШИБКА:  нет доступа к таблице t1
```
>Причина - давали права на таблицы схемы testnm, а таблица t1 у нас в схеме public, т.к. при создании не была указана конкретная схема.

```
testdb=> \dt
         Список отношений
 Схема  | Имя |   Тип   | Владелец
--------+-----+---------+----------
 public | t1  | таблица | postgres
(1 строка)
```

Подключаемся к базе пользователем postgres, удаляем таблицу t1, создаем заново с указанием схемы, вносим данные:

```
rsys@template:~$ sudo -u postgres psql -d testdb
psql (14.13 (Ubuntu 14.13-1.pgdg24.04+1))
Type "help" for help.

testdb=# DROP TABLE t1;
testdb=# CREATE TABLE testnm.t1 (c1 integer);
CREATE TABLE
testdb=# INSERT INTO testnm.t1(c1) VALUES(1);
INSERT 0 1
```

Подключаемся к базе testdb под пользователем testread и выполняем запрос:

```
rsys@template:~$ psql -h localhost -U testread -d testdb -W
Пароль:
psql (14.13 (Ubuntu 14.13-1.pgdg24.04+1))
SSL-соединение (протокол: TLSv1.3, шифр: TLS_AES_256_GCM_SHA384, бит: 256, сжатие: выкл.)
Введите "help", чтобы получить справку.

testdb=> select * from testnm.t1;
ОШИБКА:  нет доступа к таблице t1
```
>Ошибку получили, т.к. после выдачи прав мы удалили таблицу, а после создания новой права не выдавали.

Чтобы исключить такое в дальнейшем изменим права по-умолчанию на нашу схему и снова выдадим права на уже созданную таблицу:

```
rsys@template:~$ sudo -u postgres psql -d testdb
psql (14.13 (Ubuntu 14.13-1.pgdg24.04+1))
Type "help" for help.


testdb=# ALTER DEFAULT PRIVILEGES IN SCHEMA testnm GRANT SELECT ON TABLES TO readonly;
ALTER DEFAULT PRIVILEGES
testdb=# GRANT SELECT ON ALL TABLES IN SCHEMA testnm TO readonly;
GRANT
```

Для проверки создадим еще 1 таблицу и вставим данные:

```
testdb=# CREATE TABLE testnm.t2 (c1 integer);
CREATE TABLE
testdb=# INSERT INTO testnm.t2(c1) VALUES(2);
INSERT 0 1
```

Проверяем:

```
rsys@template:~$ psql -h localhost -U testread -d testdb -W
Пароль:
psql (14.13 (Ubuntu 14.13-1.pgdg24.04+1))
SSL-соединение (протокол: TLSv1.3, шифр: TLS_AES_256_GCM_SHA384, бит: 256, сжатие: выкл.)
Введите "help", чтобы получить справку.

testdb=> select * from testnm.t1;
 c1
----
  1
(1 строка)

testdb=> select * from testnm.t2;
 c1
----
  2
(1 строка)
```

Пробуем под этим пользователем создать таблицу и вставить данные:

```
testdb=> CREATE TABLE t2(c1 integer);
CREATE TABLE
testdb=> INSERT INTO t2(c1) VALUES (1);
INSERT 0 1
```
Получилось, хотя права такие не давали.
>Причину процитирую:

>Up to Postgres 14, whenever you create a database user, by default, it gets created with CREATE and USAGE privileges on the public schema.
It means that until Postgres 14, any user can write to the public schema until you manually revoke the user’s create privilege on the public schema.

Собственно этим и займемся, убираем права на создание в схеме и убираем все права во всех таблицах схемы:

```
rsys@template:~$ sudo -u postgres psql -d testdb
psql (14.13 (Ubuntu 14.13-1.pgdg24.04+1))
Type "help" for help.

testdb=# REVOKE CREATE on SCHEMA public FROM PUBLIC;
REVOKE
testdb=# REVOKE ALL ON ALL TABLES IN SCHEMA public FROM PUBLIC;
REVOKE
```

Проверяем:

```
rsys@template:~$ psql -h localhost -U testread -d testdb -W
Пароль:
psql (14.13 (Ubuntu 14.13-1.pgdg24.04+1))
SSL-соединение (протокол: TLSv1.3, шифр: TLS_AES_256_GCM_SHA384, бит: 256, сжатие: выкл.)
Введите "help", чтобы получить справку.

testdb=> CREATE TABLE t3(c1 integer);
ОШИБКА:  нет доступа к схеме public
СТРОКА 1: CREATE TABLE t3(c1 integer);
                       ^
testdb=> INSERT INTO t2(c1) VALUES (1);
INSERT 0 1
```

>Права отозваны, доступа к схеме нет. INSERT выполнился, т.к. пользователь является владельцем данной таблицы:

```
testdb=> \dt
         Список отношений
 Схема  | Имя |   Тип   | Владелец
--------+-----+---------+----------
 public | t2  | таблица | testread
(1 строка)
```