# Домашнее задание №5

## Настройка autovacuum с учетом особеностей производительности

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

Устанавливаем PostgreSQL 15:

```
sudo apt-get install postgresql-15 -y
```

Проверяем, что всё прошло успешно и запущено:

```
rsys@template:~$ pg_lsclusters
Ver Cluster Port Status Owner    Data directory              Log file
15  main    5432 online postgres /var/lib/postgresql/15/main /var/log/postgresql/postgresql-15-main.log
```

Инициализируем БД для тестов:

```
rsys@template:~$ sudo -u postgres pgbench -i postgres
dropping old tables...
ЗАМЕЧАНИЕ:  таблица "pgbench_accounts" не существует, пропускается
ЗАМЕЧАНИЕ:  таблица "pgbench_branches" не существует, пропускается
ЗАМЕЧАНИЕ:  таблица "pgbench_history" не существует, пропускается
ЗАМЕЧАНИЕ:  таблица "pgbench_tellers" не существует, пропускается
creating tables...
generating data (client-side)...
100000 of 100000 tuples (100%) done (elapsed 0.38 s, remaining 0.00 s)
vacuuming...
creating primary keys...
done in 1.02 s (drop tables 0.00 s, create tables 0.02 s, client-side generate 0.56 s, vacuum 0.16 s, primary keys 0.28 s).
```

Запускаем тестирование:

```
rsys@template:~$ sudo -u postgres pgbench -c8 -P 6 -T 60 -U postgres postgres
[sudo] password for rsys:
pgbench (15.8 (Ubuntu 15.8-1.pgdg24.04+1))
starting vacuum...end.
progress: 6.0 s, 494.5 tps, lat 15.976 ms stddev 13.182, 0 failed
progress: 12.0 s, 481.5 tps, lat 16.581 ms stddev 11.355, 0 failed
progress: 18.0 s, 714.7 tps, lat 11.209 ms stddev 8.850, 0 failed
progress: 24.0 s, 646.3 tps, lat 12.368 ms stddev 9.482, 0 failed
progress: 30.0 s, 497.3 tps, lat 16.065 ms stddev 10.738, 0 failed
progress: 36.0 s, 530.5 tps, lat 15.078 ms stddev 10.428, 0 failed
progress: 42.0 s, 505.5 tps, lat 15.811 ms stddev 10.558, 0 failed
progress: 48.0 s, 489.7 tps, lat 16.327 ms stddev 10.889, 0 failed
progress: 54.0 s, 495.2 tps, lat 16.126 ms stddev 10.899, 0 failed
progress: 60.0 s, 519.2 tps, lat 15.430 ms stddev 10.849, 0 failed
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 8
number of threads: 1
maximum number of tries: 1
duration: 60 s
number of transactions actually processed: 32254
number of failed transactions: 0 (0.000%)
latency average = 14.862 ms
latency stddev = 10.830 ms
initial connection time = 62.835 ms
tps = 537.834598 (without initial connection time)
```

Применяем предложенные в задании настройке и перезапускаем postgresql:

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


rsys@template:~$ sudo systemctl restart postgresql@15-main.service
```

Повторяем тестирование:

```
rsys@template:~$ sudo -u postgres pgbench -c8 -P 6 -T 60 -U postgres postgres
pgbench (15.8 (Ubuntu 15.8-1.pgdg24.04+1))
starting vacuum...end.
progress: 6.0 s, 416.7 tps, lat 18.978 ms stddev 13.669, 0 failed
progress: 12.0 s, 445.5 tps, lat 17.940 ms stddev 12.482, 0 failed
progress: 18.0 s, 459.2 tps, lat 17.407 ms stddev 11.725, 0 failed
progress: 24.0 s, 685.2 tps, lat 11.670 ms stddev 8.835, 0 failed
progress: 30.0 s, 638.8 tps, lat 12.521 ms stddev 9.277, 0 failed
progress: 36.0 s, 672.3 tps, lat 11.897 ms stddev 8.988, 0 failed
progress: 42.0 s, 805.2 tps, lat 9.929 ms stddev 7.682, 0 failed
progress: 48.0 s, 647.7 tps, lat 12.343 ms stddev 9.808, 0 failed
progress: 54.0 s, 484.7 tps, lat 16.475 ms stddev 11.020, 0 failed
progress: 60.0 s, 489.1 tps, lat 16.365 ms stddev 11.106, 0 failed
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 8
number of threads: 1
maximum number of tries: 1
duration: 60 s
number of transactions actually processed: 34473
number of failed transactions: 0 (0.000%)
latency average = 13.905 ms
latency stddev = 10.668 ms
initial connection time = 58.161 ms
tps = 574.838933 (without initial connection time)
```

>Получили небольшой прирост, но в пределах погрешности. На бОльших параметрах ВМ и бОльшем объеме теста разница могла бы быть больше, увеличили, хоть и немного work_mem, shared_buffers и effective_io_concurrency.

Создаем таблицу с текстовым полем и заполняем случайными данными, кол-во строк = 1млн.

```
rsys@template:~$ sudo -u postgres psql
could not change directory to "/home/rsys": Permission denied
psql (15.8 (Ubuntu 15.8-1.pgdg24.04+1))
Type "help" for help.

postgres=# CREATE TABLE testtable1(c1 char(50));
CREATE TABLE
postgres=# INSERT INTO testtable1(c1) SELECT substr(md5(random()::text), 1, 50) from generate_series(1, 1000000);
INSERT 0 1000000
```

Смотрим размер получвшейся таблицы:

```
postgres=# SELECT pg_size_pretty(pg_total_relation_size('testtable1'));
 pg_size_pretty
----------------
 81 MB
(1 row)
```

Обновляем 5 раз все строчки, добавляя по 1 символу каждый раз, смотрим размер таблицы, кол-во мертвых строк и дату последнего автовакуума:

```
postgres=# UPDATE testtable1 set c1 = '74d4b3f27497d4ada97a26bbb9';
UPDATE 1000000
postgres=# UPDATE testtable1 set c1 = '74d4b3f27497d4ada97a26bbb91';
UPDATE 1000000
postgres=# UPDATE testtable1 set c1 = '74d4b3f27497d4ada97a26bbb912';
UPDATE 1000000
postgres=# UPDATE testtable1 set c1 = '74d4b3f27497d4ada97a26bbb9123';
UPDATE 1000000
postgres=# UPDATE testtable1 set c1 = '74d4b3f27497d4ada97a26bbb91234';
UPDATE 1000000
postgres=# SELECT pg_size_pretty(pg_total_relation_size('testtable1'));
 pg_size_pretty
----------------
 483 MB
(1 row)

postgres=# SELECT relname, n_live_tup, n_dead_tup, trunc(100*n_dead_tup/(n_live_tup+1))::float "ratio%", last_autovacuum
FROM pg_stat_user_TABLES WHERE relname = 'testtable1';
  relname   | n_live_tup | n_dead_tup | ratio% |        last_autovacuum
------------+------------+------------+--------+-------------------------------
 testtable1 |    1000000 |          0 |      0 | 2024-08-27 16:41:28.304584+03
(1 row)
```

>К сожалению, автовакуум оказался быстрей меня и успел все почистить :) Тем не менее живых строк 1.000.000, а размер таблицы не ужался, т.к. не делали vacuum full.

Снова повторяем операцию по обновлению строк 5 раз и смотрим размер:

```
postgres=# UPDATE testtable1 set c1 = '74d4b3f27497d4ada97a21';
UPDATE 1000000
postgres=# UPDATE testtable1 set c1 = '74d4b3f27497d4ada97a212';
UPDATE 1000000
postgres=# UPDATE testtable1 set c1 = '74d4b3f27497d4ada97a2123';
UPDATE 1000000
postgres=# UPDATE testtable1 set c1 = '74d4b3f27497d4ada97a21234';
UPDATE 1000000
postgres=# UPDATE testtable1 set c1 = '74d4b3f27497d4ada97a212345';
UPDATE 1000000
postgres=# SELECT pg_size_pretty(pg_total_relation_size('testtable1'));
 pg_size_pretty
----------------
 483 MB
(1 row)
```

>Размер не изменился, т.к. данные были вставлены на место прошлых, очищенных автовакуумом.

Отключаем автовакуум для этой таблицы:

```
postgres=# ALTER TABLE testtable1 SET (autovacuum_enabled = off);
ALTER TABLE
```

10 раз обновляем все строчки и смотрим размер таблицы:

```
postgres=# UPDATE testtable1 set c1 = '74d4b3f27497d51';
UPDATE 1000000
postgres=# UPDATE testtable1 set c1 = '74d4b3f27497d512';
UPDATE 1000000
postgres=# UPDATE testtable1 set c1 = '74d4b3f27497d5123';
UPDATE 1000000
postgres=# UPDATE testtable1 set c1 = '74d4b3f27497d51234';
UPDATE 1000000
postgres=# UPDATE testtable1 set c1 = '74d4b3f27497d512345';
UPDATE 1000000
postgres=# UPDATE testtable1 set c1 = '74d4b3f27497d5123456';
UPDATE 1000000
postgres=# UPDATE testtable1 set c1 = '74d4b3f27497d51234567';
UPDATE 1000000
postgres=# UPDATE testtable1 set c1 = '74d4b3f27497d512345678';
UPDATE 1000000
postgres=# UPDATE testtable1 set c1 = '74d4b3f27497d5123456789';
UPDATE 1000000
postgres=# UPDATE testtable1 set c1 = '74d4b3f27497d51234567890';
UPDATE 1000000
postgres=# SELECT pg_size_pretty(pg_total_relation_size('testtable1'));
 pg_size_pretty
----------------
 886 MB
(1 row)
```

>Размер вырос, что логично, было занято все свободное пространство, образовавшееся после последнего автовакуума и начало расти дальше, т.к. данных в этот раз больше.

Включаем автовакуум обратно для очистки 10 млн лишних мертвых строк и проверяем, что он выполнился успешно:

```
postgres=# SELECT relname, n_live_tup, n_dead_tup, trunc(100*n_dead_tup/(n_live_tup+1))::float "ratio%", last_autovacuum
FROM pg_stat_user_TABLEs WHERE relname = 'testtable1';
  relname   | n_live_tup | n_dead_tup | ratio% |        last_autovacuum
------------+------------+------------+--------+-------------------------------
 testtable1 |    1000000 |    9997468 |    999 | 2024-08-27 16:50:28.693003+03
(1 row)

postgres=# SELECT relname, n_live_tup, n_dead_tup, trunc(100*n_dead_tup/(n_live_tup+1))::float "ratio%", last_autovacuum
FROM pg_stat_user_TABLEs WHERE relname = 'testtable1';
  relname   | n_live_tup | n_dead_tup | ratio% |        last_autovacuum
------------+------------+------------+--------+-------------------------------
 testtable1 |    1000000 |          0 |      0 | 2024-08-27 17:00:36.012707+03
(1 row)
```

## Задача со "звездочкой":

Написать анонимную процедуру, в которой в цикле 10 раз обновятся все строчки в искомой таблице.
Не забыть вывести номер шага цикла.


Как-то так, с этим дела никогда не имел:
```
DO $$
DECLARE
    i INTEGER;
BEGIN
    FOR i IN 1 .. 10
    LOOP
        RAISE NOTICE 'Шаг цикла = %', i;
        UPDATE testtable1 set c1 = '74d4b3f27497d4ada97a26bbb912';
    END LOOP;
END $$;
```

Проверяем, что выполнилось то, что мы хотели:

```
postgres=# DO $$
DECLARE
    i INTEGER;
BEGIN
    FOR i IN 1 .. 10
    LOOP
        RAISE NOTICE 'Шаг цикла = %', i;
        UPDATE testtable1 set c1 = '74d4b3f27497d4ada97a26bbb912';
    END LOOP;
END $$;
ЗАМЕЧАНИЕ:  Шаг цикла = 1
ЗАМЕЧАНИЕ:  Шаг цикла = 2
ЗАМЕЧАНИЕ:  Шаг цикла = 3
ЗАМЕЧАНИЕ:  Шаг цикла = 4
ЗАМЕЧАНИЕ:  Шаг цикла = 5
ЗАМЕЧАНИЕ:  Шаг цикла = 6
ЗАМЕЧАНИЕ:  Шаг цикла = 7
ЗАМЕЧАНИЕ:  Шаг цикла = 8
ЗАМЕЧАНИЕ:  Шаг цикла = 9
ЗАМЕЧАНИЕ:  Шаг цикла = 10
DO
postgres=# SELECT relname, n_live_tup, n_dead_tup, trunc(100*n_dead_tup/(n_live_tup+1))::float "ratio%", last_autovacuum
FROM pg_stat_user_TABLEs WHERE relname = 'testtable1';
  relname   | n_live_tup | n_dead_tup | ratio% |        last_autovacuum
------------+------------+------------+--------+-------------------------------
 testtable1 |    1000000 |   10000000 |    999 | 2024-08-27 17:00:36.012707+03
(1 row)
```