# Домашнее задание №6

## Работа с журналами

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

Устанавливаем PostgreSQL 16:

```
sudo apt-get install postgresql-16 -y
```

Проверяем, что всё прошло успешно и запущено:

```
rsys@template:~$ pg_lsclusters
Ver Cluster Port Status Owner    Data directory              Log file
16  main    5432 online postgres /var/lib/postgresql/16/main /var/log/postgresql/postgresql-16-main.log
```

Настроим выполнение контрольной точки каждые 30 секунд, перечитаем конфиг и проверим переменную:

```
rsys@template:~$ sudo nano /etc/postgresql/16/main/postgresql.conf
rsys@template:~$ cat /etc/postgresql/16/main/postgresql.conf |grep checkpoint_timeout
#checkpoint_timeout = 5min		# range 30s-1d
checkpoint_timeout = 30
rsys@template:~$ sudo pg_ctlcluster 16 main reload
rsys@template:~$ sudo -u postgres psql
psql (16.4 (Ubuntu 16.4-1.pgdg24.04+1))
Введите "help", чтобы получить справку.

postgres=# SHOW checkpoint_timeout;
 checkpoint_timeout
--------------------
 30s
(1 строка)
```

Подготоваливаем базу к тестировнаию и запускаем pgbench на 10 минут:

```
sudo -u postgres pgbench -i postgres
dropping old tables...
ЗАМЕЧАНИЕ:  таблица "pgbench_accounts" не существует, пропускается
ЗАМЕЧАНИЕ:  таблица "pgbench_branches" не существует, пропускается
ЗАМЕЧАНИЕ:  таблица "pgbench_history" не существует, пропускается
ЗАМЕЧАНИЕ:  таблица "pgbench_tellers" не существует, пропускается
creating tables...
generating data (client-side)...
100000 of 100000 tuples (100%) done (elapsed 0.33 s, remaining 0.00 s)
vacuuming...
creating primary keys...
done in 0.87 s (drop tables 0.00 s, create tables 0.01 s, client-side generate 0.42 s, vacuum 0.18 s, primary keys 0.25 s).

rsys@template:~$ sudo -u postgres pgbench -P 60 -T 600 postgres
pgbench (16.4 (Ubuntu 16.4-1.pgdg24.04+1))
starting vacuum...end.
progress: 60.0 s, 372.1 tps, lat 2.686 ms stddev 1.969, 0 failed
progress: 120.0 s, 362.0 tps, lat 2.762 ms stddev 0.952, 0 failed
progress: 180.0 s, 374.6 tps, lat 2.669 ms stddev 0.988, 0 failed
progress: 240.0 s, 370.8 tps, lat 2.696 ms stddev 1.022, 0 failed
progress: 300.0 s, 381.3 tps, lat 2.622 ms stddev 0.949, 0 failed
progress: 360.0 s, 383.5 tps, lat 2.606 ms stddev 0.987, 0 failed
progress: 420.0 s, 367.3 tps, lat 2.722 ms stddev 0.914, 0 failed
progress: 480.0 s, 366.3 tps, lat 2.729 ms stddev 0.924, 0 failed
progress: 540.0 s, 346.5 tps, lat 2.885 ms stddev 2.523, 0 failed
progress: 600.0 s, 342.2 tps, lat 2.921 ms stddev 2.195, 0 failed
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 1
number of threads: 1
maximum number of tries: 1
duration: 600 s
number of transactions actually processed: 220003
number of failed transactions: 0 (0.000%)
latency average = 2.726 ms
latency stddev = 1.454 ms
initial connection time = 6.719 ms
tps = 366.673540 (without initial connection time)
```

Измеряем объём созданных за это время журнальных файлов:

```
rsys@template:~$ sudo -u postgres du -sm /var/lib/postgresql/16/main/pg_wal
65	/var/lib/postgresql/16/main/pg_wal
```
>В среднем на 1 контрольную точку получаем 65 Мб / 20 чекпоинтов = 3.25Мб.

Проверяем все ли контрольные точки выполнились по расписанию:

```
postgres=# SELECT * FROM pg_stat_bgwriter \gx
-[ RECORD 1 ]---------+------------------------------
checkpoints_timed     | 57
checkpoints_req       | 0
checkpoint_write_time | 596408
checkpoint_sync_time  | 238
buffers_checkpoint    | 40387
buffers_clean         | 0
maxwritten_clean      | 0
buffers_backend       | 1549
buffers_backend_fsync | 0
buffers_alloc         | 3915
stats_reset           | 2024-08-30 12:32:07.055646+03
```
>На данный момент все чекпоинты выполнились по расписанию.

Сравним показатель tps в синхронном и асинхронном режиме утилитой pgbench:

```
postgres=# show synchronous_commit;
 synchronous_commit
--------------------
 on
(1 строка)

rsys@template:~$ sudo -u postgres pgbench -P 1 -T 10 postgres
pgbench (16.4 (Ubuntu 16.4-1.pgdg24.04+1))
starting vacuum...end.
progress: 1.0 s, 499.0 tps, lat 1.984 ms stddev 0.829, 0 failed
progress: 2.0 s, 308.0 tps, lat 3.239 ms stddev 1.096, 0 failed
progress: 3.0 s, 313.0 tps, lat 3.198 ms stddev 0.618, 0 failed
progress: 4.0 s, 300.0 tps, lat 3.336 ms stddev 0.765, 0 failed
progress: 5.0 s, 303.0 tps, lat 3.286 ms stddev 0.694, 0 failed
progress: 6.0 s, 320.0 tps, lat 3.131 ms stddev 0.643, 0 failed
progress: 7.0 s, 325.0 tps, lat 3.074 ms stddev 0.541, 0 failed
progress: 8.0 s, 334.0 tps, lat 2.989 ms stddev 0.760, 0 failed
progress: 9.0 s, 314.0 tps, lat 3.183 ms stddev 0.770, 0 failed
progress: 10.0 s, 344.0 tps, lat 2.912 ms stddev 0.409, 0 failed
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 1
number of threads: 1
maximum number of tries: 1
duration: 10 s
number of transactions actually processed: 3361
number of failed transactions: 0 (0.000%)
latency average = 2.972 ms
latency stddev = 0.852 ms
initial connection time = 9.328 ms
tps = 336.327660 (without initial connection time)

rsys@template:~$ sudo -u postgres psql
psql (16.4 (Ubuntu 16.4-1.pgdg24.04+1))
Введите "help", чтобы получить справку.

postgres=# alter system set synchronous_commit = off;
ALTER SYSTEM
postgres=# select pg_reload_conf();
 pg_reload_conf
----------------
 t
(1 строка)

postgres=# show synchronous_commit;
 synchronous_commit
--------------------
 off
(1 строка)

rsys@template:~$ sudo -u postgres pgbench -P 1 -T 10 postgres
pgbench (16.4 (Ubuntu 16.4-1.pgdg24.04+1))
starting vacuum...end.
progress: 1.0 s, 769.0 tps, lat 1.278 ms stddev 0.575, 0 failed
progress: 2.0 s, 597.9 tps, lat 1.671 ms stddev 0.732, 0 failed
progress: 3.0 s, 503.1 tps, lat 1.986 ms stddev 0.459, 0 failed
progress: 4.0 s, 461.0 tps, lat 2.170 ms stddev 0.141, 0 failed
progress: 5.0 s, 597.0 tps, lat 1.674 ms stddev 0.719, 0 failed
progress: 6.0 s, 522.0 tps, lat 1.913 ms stddev 0.425, 0 failed
progress: 7.0 s, 465.0 tps, lat 2.149 ms stddev 0.116, 0 failed
progress: 8.0 s, 462.0 tps, lat 2.164 ms stddev 0.513, 0 failed
progress: 9.0 s, 835.0 tps, lat 1.198 ms stddev 0.450, 0 failed
progress: 10.0 s, 461.9 tps, lat 2.165 ms stddev 0.312, 0 failed
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 1
number of threads: 1
maximum number of tries: 1
duration: 10 s
number of transactions actually processed: 5675
number of failed transactions: 0 (0.000%)
latency average = 1.759 ms
latency stddev = 0.627 ms
initial connection time = 16.243 ms
tps = 568.077053 (without initial connection time)
```

>В синхронном режиме получили значение 336 tps, в асинхронном 568 tps. Причина - в асинхронном режиме отсутствует ожидании локального сброса WAL на диск.

Создаем новый кластер с включенной контрольной суммой страниц:

```
rsys@template:~$ sudo -u postgres pg_createcluster 16 chksumcluster -- --data-checksums
Creating new PostgreSQL cluster 16/chksumcluster ...
/usr/lib/postgresql/16/bin/initdb -D /var/lib/postgresql/16/chksumcluster --auth-local peer --auth-host scram-sha-256 --no-instructions --data-checksums
Файлы, относящиеся к этой СУБД, будут принадлежать пользователю "postgres".
От его имени также будет запускаться процесс сервера.

Кластер баз данных будет инициализирован с локалью "ru_RU.UTF-8".
Кодировка БД по умолчанию, выбранная в соответствии с настройками: "UTF8".
Выбрана конфигурация текстового поиска по умолчанию "russian".

Контроль целостности страниц данных включён.

исправление прав для существующего каталога /var/lib/postgresql/16/chksumcluster... ок
создание подкаталогов... ок
выбирается реализация динамической разделяемой памяти... posix
выбирается значение max_connections по умолчанию... 100
выбирается значение shared_buffers по умолчанию... 128MB
выбирается часовой пояс по умолчанию... Europe/Moscow
создание конфигурационных файлов... ок
выполняется подготовительный скрипт... ок
выполняется заключительная инициализация... ок
сохранение данных на диске... ок
Warning: systemd does not know about the new cluster yet. Operations like "service postgresql start" will not handle it. To fix, run:
  sudo systemctl daemon-reload
Ver Cluster       Port Status Owner    Data directory                       Log file
16  chksumcluster 5433 down   postgres /var/lib/postgresql/16/chksumcluster /var/log/postgresql/postgresql-16-chksumcluster.log

rsys@template:~$ sudo systemctl daemon-reload
rsys@template:~$ sudo systemctl start postgresql@16-chksumcluster
rsys@template:~$ pg_lsclusters
Ver Cluster       Port Status Owner    Data directory                       Log file
16  chksumcluster 5433 online postgres /var/lib/postgresql/16/chksumcluster /var/log/postgresql/postgresql-16-chksumcluster.log
16  main          5432 online postgres /var/lib/postgresql/16/main          /var/log/postgresql/postgresql-16-main.log

```

Подключаемся к данному кластеру и проверяем, что контроль целостности включен:

```
rsys@template:~$ sudo -u postgres psql -p5433
psql (16.4 (Ubuntu 16.4-1.pgdg24.04+1))
Введите "help", чтобы получить справку.

postgres=# show data_checksums;
 data_checksums
----------------
 on
(1 строка)
```

Создаем таблицу, заполняем данными:

```
postgres=# CREATE TABLE t1 (c1 integer);
CREATE TABLE
postgres=# INSERT INTO t1(c1) VALUES(1),(2),(3);
INSERT 0 3
```

Смотрим путь до таблички, останавливаем данный кластер и "портим" файл с таблицей:

```
postgres=# select pg_relation_filepath('t1');
 pg_relation_filepath
----------------------
 base/5/16388
(1 строка)

rsys@template:~$ sudo systemctl stop postgresql@16-chksumcluster

rsys@template:~$ sudo dd if=/dev/zero of=/var/lib/postgresql/16/chksumcluster/base/5/16388 oflag=dsync conv=notrunc bs=1 count=8
8+0 records in
8+0 records out
8 bytes copied, 0,00385767 s, 2,1 kB/s
```

Стартуем кластер, подключаемся и пробуем сделать выборку из таблицы:

```
rsys@template:~$ sudo systemctl start postgresql@16-chksumcluster
rsys@template:~$ sudo -u postgres psql -p5433
psql (16.4 (Ubuntu 16.4-1.pgdg24.04+1))
Введите "help", чтобы получить справку.

postgres=# SELECT * FROM t1;
ПРЕДУПРЕЖДЕНИЕ:  ошибка проверки страницы: получена контрольная сумма 32630, а ожидалась - 42284
ОШИБКА:  неверная страница в блоке 0 отношения base/5/16388
```
>В результате нашего вмешательства изменилась контрольная сумма, получили ошибку. Если необходимо игнорировать ошибку, мы можем воспользоваться параметром ignore_checksum_failure и попробовать прочитать уцелевшие данные:

```
postgres=# SET ignore_checksum_failure = true;
SET
postgres=# show ignore_checksum_failure;
 ignore_checksum_failure
-------------------------
 on
(1 строка)

postgres=# SELECT * FROM t1;
ПРЕДУПРЕЖДЕНИЕ:  ошибка проверки страницы: получена контрольная сумма 32630, а ожидалась - 42284
 c1
----
  1
  2
  3
(3 строки)
```

>Предупреждение в таком случае все равно получаем, но и данные смогли прочитать.