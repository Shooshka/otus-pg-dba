# Домашнее задание №9

## Механизм блокировок

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
rsys@template:~$ sudo pg_lsclusters
Ver Cluster Port Status Owner    Data directory              Log file
16  main    5432 online postgres /var/lib/postgresql/16/main /var/log/postgresql/postgresql-16-main.log
```

Настроим логирование сообщений, которые содержат информацию о блокировках, удерживающихся более 200мс и перечитаем конфиг postgresql:

```
postgres@template:/home/rsys$ echo 'log_lock_waits = on' >> /etc/postgresql/16/main/postgresql.conf
postgres@template:/home/rsys$ echo 'deadlock_timeout = '200ms'' >> /etc/postgresql/16/main/postgresql.conf
rsys@template:~$ sudo systemctl reload postgresql@16-main.service
```

Создадим таблицу и наполним данными:

```
rsys@template:~$ sudo -u postgres psql
psql (16.4 (Ubuntu 16.4-1.pgdg24.04+1))
Введите "help", чтобы получить справку.

postgres=# CREATE TABLE table1 as select generate_series(1, 10) as id, md5(random()::text)::char(5) as rndtext;
SELECT 10
```

Запустим еще 2 сессии psql и во всех 3 сеансах запустим UPDATE одной и той же строки:

```
1: 

postgres=# BEGIN;
BEGIN
postgres=*# UPDATE table1 SET rndtext = 'rnd12' WHERE id = 1;
UPDATE 1

2:
postgres=# BEGIN;
BEGIN
postgres=*# UPDATE table1 SET rndtext = 'rnd12' WHERE id = 1;

3:
postgres=# BEGIN;
BEGIN
postgres=*# UPDATE table1 SET rndtext = 'rnd12' WHERE id = 1;
```
>Видим, что во 2 и 3 сеансе транзакция повисла.

Посмотрим на блокировки нашей таблицы:

```
postgres=*# SELECT locktype, mode, granted, pid, pg_blocking_pids(pid) AS wait_for FROM pg_locks WHERE relation = 'table1'::regclass;
 locktype |       mode       | granted | pid  | wait_for
----------+------------------+---------+------+----------
 relation | RowExclusiveLock | t       | 7938 | {7931}
 relation | RowExclusiveLock | t       | 7931 | {7801}
 relation | RowExclusiveLock | t       | 7801 | {}
 tuple    | ExclusiveLock    | t       | 7931 | {7801}
 tuple    | ExclusiveLock    | f       | 7938 | {7931}
(5 строк)
```

>Видим, что на блокировках отношения/таблицы 1 сеанс с пидом 7801 является первым, кто получил блокировку, 2 сеанс с пидом 7931 ожидает освобождения от 7801 и третий сеанс с пидом 7938 ожидает освобождения от 7931. Аналогично и на блокировке самой строки, блокировка грантед на второй пид, ожидает завершения первого пида, и пока еще не грантед на третий пид, ожидая освобождения от второго пида.

Если посмотреть в журнал, то увидим в логе сообщения о блокировках, настроенные ранее:

```
2024-09-13 10:46:54.521 MSK [7938] postgres@postgres СООБЩЕНИЕ:  процесс 7938 получил в режиме ShareLock блокировку "транзакция 743" через 31262.666 мс
2024-09-13 10:46:54.521 MSK [7938] postgres@postgres КОНТЕКСТ:  при изменении кортежа (0,1) в отношении "table1"
2024-09-13 10:46:54.521 MSK [7938] postgres@postgres ОПЕРАТОР:  UPDATE table1 SET rndtext = 'rnd12' WHERE id = 1;
2024-09-13 10:47:20.332 MSK [7801] postgres@postgres ПРЕДУПРЕЖДЕНИЕ:  нет незавершённой транзакции
2024-09-13 10:47:31.843 MSK [7931] postgres@postgres СООБЩЕНИЕ:  процесс 7931 продолжает ожидать в режиме ShareLock блокировку "транзакция 747" в течение 200.351 мс
2024-09-13 10:47:31.843 MSK [7931] postgres@postgres ПОДРОБНОСТИ:  Process holding the lock: 7801. Wait queue: 7931.
2024-09-13 10:47:31.843 MSK [7931] postgres@postgres КОНТЕКСТ:  при изменении кортежа (0,15) в отношении "table1"
2024-09-13 10:47:31.843 MSK [7931] postgres@postgres ОПЕРАТОР:  UPDATE table1 SET rndtext = 'rnd12' WHERE id = 1;
2024-09-13 10:47:37.903 MSK [7938] postgres@postgres СООБЩЕНИЕ:  процесс 7938 продолжает ожидать в режиме ExclusiveLock блокировку "кортеж (0,15) отношения 16388 базы данных 5" в течение 200.359 мс
```

Сделаем rollback/commit всех транзакций во всех сеансах и перейдем к следующему пункту ДЗ.

Воспроизведем взаимоблокировку трех транзакций:

```
Сессия 1:
postgres=# BEGIN;
BEGIN
postgres=*# UPDATE table1 SET rndtext = '123' WHERE id = 1;
UPDATE 1

Сессия 2:
postgres=# BEGIN;
BEGIN
postgres=*# UPDATE table1 SET rndtext = '123' WHERE id = 2;
UPDATE 1

Сессия 3:
postgres=# BEGIN;
BEGIN
postgres=*# UPDATE table1 SET rndtext = '123' WHERE id = 3;
UPDATE 1

Сессия 1 (зависает, т.к. ждет транзакцию в сеансе 2):
postgres=*# UPDATE table1 SET rndtext = '123' WHERE id = 2;

Сессия 2 (зависает, т.к. ждет транзакцию в сеансе 3):
postgres=*# UPDATE table1 SET rndtext = '123' WHERE id = 3;

Сессия 3 (Зависает, но тут же обрывается с ошибкой, т.к. происходит дедлок):
postgres=*# UPDATE table1 SET rndtext = '123' WHERE id = 1;
ОШИБКА:  обнаружена взаимоблокировка
ПОДРОБНОСТИ:  Процесс 7938 ожидает в режиме ShareLock блокировку "транзакция 750"; заблокирован процессом 8006.
Процесс 8006 ожидает в режиме ShareLock блокировку "транзакция 751"; заблокирован процессом 7931.
Процесс 7931 ожидает в режиме ShareLock блокировку "транзакция 752"; заблокирован процессом 7938.
ПОДСКАЗКА:  Подробности запроса смотрите в протоколе сервера.
КОНТЕКСТ:  при изменении кортежа (0,15) в отношении "table1"
```
>Разобраться постфактум можно, смотрим в лог сервера и видим весь процесс:

```
2024-09-13 11:00:13.783 MSK [8006] postgres@postgres СООБЩЕНИЕ:  процесс 8006 продолжает ожидать в режиме ShareLock блокировку "транзакция 751" в течение 200.987 мс
2024-09-13 11:00:13.783 MSK [8006] postgres@postgres ПОДРОБНОСТИ:  Process holding the lock: 7931. Wait queue: 8006.
2024-09-13 11:00:13.783 MSK [8006] postgres@postgres КОНТЕКСТ:  при изменении кортежа (0,2) в отношении "table1"
2024-09-13 11:00:13.783 MSK [8006] postgres@postgres ОПЕРАТОР:  UPDATE table1 SET rndtext = '123' WHERE id = 2;
2024-09-13 11:00:17.820 MSK [7931] postgres@postgres СООБЩЕНИЕ:  процесс 7931 продолжает ожидать в режиме ShareLock блокировку "транзакция 752" в течение 200.334 мс
2024-09-13 11:00:17.820 MSK [7931] postgres@postgres ПОДРОБНОСТИ:  Process holding the lock: 7938. Wait queue: 7931.
2024-09-13 11:00:17.820 MSK [7931] postgres@postgres КОНТЕКСТ:  при изменении кортежа (0,3) в отношении "table1"
2024-09-13 11:00:17.820 MSK [7931] postgres@postgres ОПЕРАТОР:  UPDATE table1 SET rndtext = '123' WHERE id = 3;
2024-09-13 11:00:22.330 MSK [7938] postgres@postgres СООБЩЕНИЕ:  процесс 7938 обнаружил взаимоблокировку, ожидая в режиме ShareLock блокировку "транзакция 750" в течение 200.355 мс
2024-09-13 11:00:22.330 MSK [7938] postgres@postgres ПОДРОБНОСТИ:  Process holding the lock: 8006. Wait queue: .
2024-09-13 11:00:22.330 MSK [7938] postgres@postgres КОНТЕКСТ:  при изменении кортежа (0,15) в отношении "table1"
2024-09-13 11:00:22.330 MSK [7938] postgres@postgres ОПЕРАТОР:  UPDATE table1 SET rndtext = '123' WHERE id = 1;
2024-09-13 11:00:22.331 MSK [7938] postgres@postgres ОШИБКА:  обнаружена взаимоблокировка
2024-09-13 11:00:22.331 MSK [7938] postgres@postgres ПОДРОБНОСТИ:  Процесс 7938 ожидает в режиме ShareLock блокировку "транзакция 750"; заблокирован процессом 8006.
	Процесс 8006 ожидает в режиме ShareLock блокировку "транзакция 751"; заблокирован процессом 7931.
	Процесс 7931 ожидает в режиме ShareLock блокировку "транзакция 752"; заблокирован процессом 7938.
	Процесс 7938: UPDATE table1 SET rndtext = '123' WHERE id = 1;
	Процесс 8006: UPDATE table1 SET rndtext = '123' WHERE id = 2;
	Процесс 7931: UPDATE table1 SET rndtext = '123' WHERE id = 3;
2024-09-13 11:00:22.331 MSK [7938] postgres@postgres ПОДСКАЗКА:  Подробности запроса смотрите в протоколе сервера.
2024-09-13 11:00:22.331 MSK [7938] postgres@postgres КОНТЕКСТ:  при изменении кортежа (0,15) в отношении "table1"
2024-09-13 11:00:22.331 MSK [7938] postgres@postgres ОПЕРАТОР:  UPDATE table1 SET rndtext = '123' WHERE id = 1;
2024-09-13 11:00:22.332 MSK [7931] postgres@postgres СООБЩЕНИЕ:  процесс 7931 получил в режиме ShareLock блокировку "транзакция 752" через 4712.783 мс
2024-09-13 11:00:22.332 MSK [7931] postgres@postgres КОНТЕКСТ:  при изменении кортежа (0,3) в отношении "table1"
2024-09-13 11:00:22.332 MSK [7931] postgres@postgres ОПЕРАТОР:  UPDATE table1 SET rndtext = '123' WHERE id = 3;
```

Переходим к следующему пункту ДЗ - "Могут ли две транзакции, выполняющие единственную команду UPDATE одной и той же таблицы (без where), заблокировать друг друга?"

>Могут, если в 1сеансе апдейт будет выполнятся в одном порядке, а во втором сеансе в другом.

## Задание со * - воспроизвести такую ситуацию.

Возьмем пример из книги и воспроизведем его:

```
postgres=# CREATE TABLE accounts(
id integer PRIMARY KEY GENERATED BY DEFAULT AS IDENTITY, client text,
amount numeric
);
CREATE TABLE

postgres=# INSERT INTO accounts(id, client, amount) VALUES (1,'alice',100.00),
(2,'bob',200.00),
(3,'charlie',300.00);
INSERT 0 3

postgres=# CREATE INDEX ON accounts(amount DESC);
CREATE INDEX

postgres=# CREATE FUNCTION inc_slow(n numeric) RETURNS numeric
AS $$
SELECT pg_sleep(1);
SELECT n + 100.00; $$ LANGUAGE sql;
CREATE FUNCTION

1 сеанс:
postgres=# UPDATE accounts SET amount = inc_slow(amount);

2 сеанс:
postgres=# SET enable_seqscan = off;
SET
postgres=# UPDATE accounts SET amount = inc_slow(amount) WHERE amount > 100.00;

И во 2 сеансе получаем:

ОШИБКА:  обнаружена взаимоблокировка
ПОДРОБНОСТИ:  Процесс 7931 ожидает в режиме ShareLock блокировку "транзакция 759"; заблокирован процессом 8006.
Процесс 8006 ожидает в режиме ShareLock блокировку "транзакция 760"; заблокирован процессом 7931.
ПОДСКАЗКА:  Подробности запроса смотрите в протоколе сервера.
КОНТЕКСТ:  при изменении кортежа (0,4) в отношении "accounts"
```

Убедились в возможности такой ситуации.