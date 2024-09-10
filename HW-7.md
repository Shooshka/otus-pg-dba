# Домашнее задание №7

## Нагрузочное тестирование и тюнинг PostgreSQL

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
rsys@template:~$ sudo pg_lsclusters
Ver Cluster Port Status Owner    Data directory              Log file
15  main    5432 online postgres /var/lib/postgresql/15/main /var/log/postgresql/postgresql-15-main.log
```

Инициализируем БД для тестов и выполняем тест:

```
rsys@template:~$ sudo -u postgres pgbench -i postgres
dropping old tables...
ЗАМЕЧАНИЕ:  таблица "pgbench_accounts" не существует, пропускается
ЗАМЕЧАНИЕ:  таблица "pgbench_branches" не существует, пропускается
ЗАМЕЧАНИЕ:  таблица "pgbench_history" не существует, пропускается
ЗАМЕЧАНИЕ:  таблица "pgbench_tellers" не существует, пропускается
creating tables...
generating data (client-side)...
100000 of 100000 tuples (100%) done (elapsed 0.16 s, remaining 0.00 s)
vacuuming...
creating primary keys...
done in 0.66 s (drop tables 0.00 s, create tables 0.01 s, client-side generate 0.25 s, vacuum 0.13 s, primary keys 0.27 s).

rsys@template:~$ sudo -u postgres pgbench -c 10 -j 2 -P 6 -T 30 -U postgres postgres
pgbench (15.8 (Ubuntu 15.8-1.pgdg24.04+1))
starting vacuum...end.
progress: 6.0 s, 514.3 tps, lat 19.274 ms stddev 13.663, 0 failed
progress: 12.0 s, 547.5 tps, lat 18.283 ms stddev 13.769, 0 failed
progress: 18.0 s, 722.7 tps, lat 13.831 ms stddev 11.786, 0 failed
progress: 24.0 s, 535.2 tps, lat 18.673 ms stddev 14.576, 0 failed
progress: 30.0 s, 676.7 tps, lat 14.774 ms stddev 12.134, 0 failed
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 10
number of threads: 2
maximum number of tries: 1
duration: 30 s
number of transactions actually processed: 17988
number of failed transactions: 0 (0.000%)
latency average = 16.663 ms
latency stddev = 13.289 ms
initial connection time = 35.878 ms
tps = 599.596522 (without initial connection time)
```

Применим значения, которые посоветует конфигуратор для нашей ВМ, при этом соглашаемся с пунктами "Can you lose single transactions in case of a crash?" и "Are you willing to try out experimental features for better performance?", т.к. нам важно получить бОльшее значение tps.

```
# DISCLAIMER - Software and the resulting config files are provided AS IS - IN NO EVENT SHALL
# BE THE CREATOR LIABLE TO ANY PARTY FOR DIRECT, INDIRECT, SPECIAL, INCIDENTAL, OR CONSEQUENTIAL
# DAMAGES, INCLUDING LOST PROFITS, ARISING OUT OF THE USE OF THIS SOFTWARE AND ITS DOCUMENTATION.

# Connectivity
max_connections = 20
superuser_reserved_connections = 3

# Memory Settings
shared_buffers = '1024 MB'
work_mem = '32 MB'
maintenance_work_mem = '320 MB'
huge_pages = off
effective_cache_size = '3 GB'
effective_io_concurrency = 100 # concurrent IO only really activated if OS supports posix_fadvise function
random_page_cost = 1.25 # speed of random disk access relative to sequential access (1.0)

# Monitoring
shared_preload_libraries = 'pg_stat_statements'    # per statement resource usage stats
track_io_timing=on        # measure exact block IO times
track_functions=pl        # track execution times of pl-language procedures if any

# Replication
wal_level = replica		# consider using at least 'replica'
max_wal_senders = 0
synchronous_commit = off

# Checkpointing: 
checkpoint_timeout  = '15 min' 
checkpoint_completion_target = 0.9
max_wal_size = '1024 MB'
min_wal_size = '512 MB'


# WAL writing
wal_compression = on
wal_buffers = -1    # auto-tuned by Postgres till maximum of segment size (16MB by default)


# Background writer
bgwriter_delay = 200ms
bgwriter_lru_maxpages = 100
bgwriter_lru_multiplier = 2.0
bgwriter_flush_after = 0

# Parallel queries: 
max_worker_processes = 2
max_parallel_workers_per_gather = 1
max_parallel_maintenance_workers = 1
max_parallel_workers = 2
parallel_leader_participation = on

# Advanced features 
enable_partitionwise_join = on 
enable_partitionwise_aggregate = on
jit = on
max_slot_wal_keep_size = '1000 MB'
track_wal_io_timing = on
maintenance_io_concurrency = 100
wal_recycle = on


# General notes:
# Note that not all settings are automatically tuned.
#   Consider contacting experts at 
#   https://www.cybertec-postgresql.com 
#   for more professional expertise.

```

После внесения изменений в конфиг перезагружаем постгрес и выполняем тест с такими же параметрами:

```
rsys@template:~$ sudo systemctl restart postgresql@15-main.service

rsys@template:~$ sudo -u postgres pgbench -c 10 -j 2 -P 6 -T 30 -U postgres postgres
pgbench (15.8 (Ubuntu 15.8-1.pgdg24.04+1))
starting vacuum...end.
progress: 6.0 s, 1716.1 tps, lat 5.777 ms stddev 3.435, 0 failed
progress: 12.0 s, 1808.5 tps, lat 5.527 ms stddev 3.359, 0 failed
progress: 18.0 s, 1779.7 tps, lat 5.616 ms stddev 3.647, 0 failed
progress: 24.0 s, 1784.8 tps, lat 5.600 ms stddev 3.370, 0 failed
progress: 30.0 s, 1815.9 tps, lat 5.505 ms stddev 3.317, 0 failed
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 10
number of threads: 2
maximum number of tries: 1
duration: 30 s
number of transactions actually processed: 53436
number of failed transactions: 0 (0.000%)
latency average = 5.604 ms
latency stddev = 3.430 ms
initial connection time = 48.071 ms
tps = 1782.813625 (without initial connection time)
```

>Получили прирост производительности примерно в 3 раза, в основном из-за параметра synchronous_commit = off.

# Задание со *

## Тестируем утилитой sysbench

Установим утилиту sysbench и проверим, что все успешно:

```
rsys@template:~$ curl -s https://packagecloud.io/install/repositories/akopytov/sysbench/script.deb.sh | sudo bash sudo apt -y install sysbench

rsys@template:~$ sysbench --version
sysbench 1.0.20
```

Клонируем репозиторий sysbench-tpcc и перейдем в каталог с исполняемыми файлами (качать из релизов не стоит, старая версия и работать не будет возможно с текущими версиями постгрес):

```
rsys@template:~/sysbench$ git clone https://github.com/Percona-Lab/sysbench-tpcc
rsys@template:~/sysbench$ cd sysbench-tpcc/
```

Подготовим базу в postgresql:

```
postgres=# CREATE DATABASE sbtest;
CREATE DATABASE
```

Подготовим БД с помощью sysbench-tpcc:

```
rsys@template:~/sysbench$ ./tpcc.lua --pgsql-user=postgres --pgsql-password=postgres --pgsql-db=sbtest --time=30 --threads=10 --report-interval=5 --tables=1 --scale=10 --use_fk=0 --trx_level=RC --db-driver=pgsql prepare

```

После успешного наполнения данными БД запускаем сам тест:

```
rsys@template:~/sysbench/sysbench-tpcc$ ./tpcc.lua --pgsql-user=postgres --pgsql-password=postgres --pgsql-db=sbtest --time=30 --threads=10 --report-interval=5 --tables=1 --scale=10 --use_fk=0 --trx_level=RC --db-driver=pgsql --db-ps-mode=auto run

---

[ 5s ] thds: 10 tps: 177.33 qps: 5084.96 (r/w/o: 2320.71/2403.59/360.66) lat (ms,95%): 150.29 err/s 1.40 reconn/s: 0.00
[ 10s ] thds: 10 tps: 237.57 qps: 6955.98 (r/w/o: 3180.53/3300.31/475.13) lat (ms,95%): 99.33 err/s 1.00 reconn/s: 0.00
[ 15s ] thds: 10 tps: 261.42 qps: 7449.93 (r/w/o: 3408.04/3519.05/522.84) lat (ms,95%): 97.55 err/s 1.40 reconn/s: 0.00
[ 20s ] thds: 10 tps: 254.02 qps: 7605.10 (r/w/o: 3472.63/3624.44/508.03) lat (ms,95%): 92.42 err/s 1.40 reconn/s: 0.00
[ 25s ] thds: 10 tps: 274.82 qps: 7691.77 (r/w/o: 3501.46/3640.67/549.64) lat (ms,95%): 89.16 err/s 1.40 reconn/s: 0.00
[ 30s ] thds: 10 tps: 278.80 qps: 8003.88 (r/w/o: 3657.04/3789.44/557.41) lat (ms,95%): 87.56 err/s 1.20 reconn/s: 0.00
SQL statistics:
    queries performed:
        read:                            97777
        write:                           101470
        other:                           14880
        total:                           214127
    transactions:                        7430   (247.16 per sec.)
    queries:                             214127 (7122.98 per sec.)
    ignored errors:                      39     (1.30 per sec.)
    reconnects:                          0      (0.00 per sec.)
```

>Получили 247tps и 7122qps.

Теперь убираем из конфига postgresql рекомендуемые параметры, перегружаем postgresql и запускаем тест повторно, ожидаем падение производительности:

```
rsys@template:~/sysbench/sysbench-tpcc$ ./tpcc.lua --pgsql-user=postgres --pgsql-password=postgres --pgsql-db=sbtest --time=30 --threads=10 --report-interval=5 --tables=1 --scale=10 --use_fk=0 --trx_level=RC --db-driver=pgsql --db-ps-mode=auto run

[ 5s ] thds: 10 tps: 147.91 qps: 4198.37 (r/w/o: 1923.23/1973.33/301.81) lat (ms,95%): 211.60 err/s 1.00 reconn/s: 0.00
[ 10s ] thds: 10 tps: 182.74 qps: 5117.53 (r/w/o: 2344.00/2408.05/365.48) lat (ms,95%): 144.97 err/s 0.40 reconn/s: 0.00
[ 15s ] thds: 10 tps: 218.46 qps: 6125.85 (r/w/o: 2788.40/2900.53/436.92) lat (ms,95%): 112.67 err/s 1.00 reconn/s: 0.00
[ 20s ] thds: 10 tps: 252.79 qps: 6961.54 (r/w/o: 3177.84/3278.11/505.59) lat (ms,95%): 94.10 err/s 1.80 reconn/s: 0.00
[ 25s ] thds: 10 tps: 243.31 qps: 7199.20 (r/w/o: 3285.52/3427.07/486.61) lat (ms,95%): 95.81 err/s 1.80 reconn/s: 0.00
[ 30s ] thds: 10 tps: 250.85 qps: 7069.87 (r/w/o: 3224.47/3343.90/501.50) lat (ms,95%): 97.55 err/s 1.80 reconn/s: 0.00
SQL statistics:
    queries performed:
        read:                            83825
        write:                           86772
        other:                           13000
        total:                           183597
    transactions:                        6490   (215.75 per sec.)
    queries:                             183597 (6103.27 per sec.)
    ignored errors:                      39     (1.30 per sec.)
    reconnects:                          0      (0.00 per sec.)
```

Получили значения ниже предыдущих (с настройками на бОльшую производительность), что и ожидалось.