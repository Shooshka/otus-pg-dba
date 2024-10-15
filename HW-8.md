# Домашнее задание №8

## Бэкапы

### Данные о сервере

Для работы используется виртуальная машина на частном облаке со следующими характеристиками:

* CPU: 2 core Xeon E5-2690v2
* RAM: 4 Gb
* SSD: 20 Gb SSD
* ОС: Ubuntu 24.04 LTS

### Обновление ОС и установка необходимого ПО

Запускаем полное обновление ОС перед началом работы:

```bash
sudo apt-get update && sudo apt-get dist-upgrade -y
```

После окончания обновления при необходимости перезагружаем ВМ:

```bash
sudo reboot
```

Подключаем репозиторий PostgreSQL:

```bash
sudo apt install -y postgresql-common
sudo /usr/share/postgresql-common/pgdg/apt.postgresql.org.sh
```

Устанавливаем PostgreSQL 16:

```bash
sudo apt-get install postgresql-16 -y
```

Проверяем, что всё прошло успешно и запущено:

```bash
rsys@template:~$ sudo pg_lsclusters
Ver Cluster Port Status Owner    Data directory              Log file
16  main    5432 online postgres /var/lib/postgresql/16/main /var/log/postgresql/postgresql-16-main.log
```

Подключимся, создадим БД, схему, таблицу и наполним данными в 100 строк:

```postgresql
postgres=# CREATE DATABASE base1;
CREATE DATABASE
postgres=# \c base1
Вы подключены к базе данных "base1" как пользователь "postgres".
base1=# CREATE SCHEMA our_schema;
CREATE SCHEMA
base1=# CREATE TABLE our_schema.our_table as select generate_series(1, 100) as id, md5(random()::text)::char(5) as rndtext;
SELECT 100
```

Создаем каталог для бэкапов под пользователем postgres:

```bash
rsys@template:~$ sudo -u postgres mkdir /tmp/backups
```

Сделаем логический бэкап используя утилиту COPY:

```postgresql
base1=# \copy our_schema.our_table to '/tmp/backups/logical_copy.sql';
COPY 100
```

Восстановим во 2 таблицу данные из бэкапа и проверим наличие данных:

```postgresql
base1=# CREATE TABLE our_schema.our_table2 as select generate_series(1, 100) as id, md5(random()::text)::char(5) as rndtext;
SELECT 100
base1=# TRUNCATE our_schema.our_table2;
TRUNCATE TABLE
base1=# SELECT * FROM our_schema.our_table2;
 id | rndtext
----+---------
(0 строк)

base1=# \copy our_schema.our_table2 from '/tmp/backups/logical_copy.sql';
COPY 100
base1=# SELECT * FROM our_schema.our_table2 LIMIT 5;
 id | rndtext
----+---------
  1 | 56017
  2 | 06751
  3 | f9a84
  4 | 8bc8a
  5 | 4e41e
(5 строк)
```

Используя утилиту pg_dump создадим бэкап в кастомном сжатом формате двух таблиц:

```bash
postgres@template:/home/rsys$ pg_dump -d base1 --create -Fc > /tmp/backups/dump.gz
```

Используя утилиту pg_restore восстановим в новую БД только вторую таблицу:

```bash
createdb new_db && pg_restore -U postgres --schema-only -d new_db /tmp/backups/dump.gz
pg_restore -U postgres --data-only -d new_db -t our_table2 /tmp/backups/dump.gz

postgres@template:/home/rsys$ psql
psql (16.4 (Ubuntu 16.4-1.pgdg24.04+1))
Введите "help", чтобы получить справку.
```

```postgresql
postgres=# \c new_db
Вы подключены к базе данных "new_db" как пользователь "postgres".
new_db=# select * from our_schema.our_table limit 10;
 id | rndtext
----+---------
(0 строк)

new_db=# select * from our_schema.our_table2 limit 10;
 id | rndtext
----+---------
  1 | 56017
  2 | 06751
  3 | f9a84
  4 | 8bc8a
  5 | 4e41e
  6 | 40d45
  7 | e21c4
  8 | b3faa
  9 | 10864
 10 | 49718
(10 строк)
```
