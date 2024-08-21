# Домашнее задание №3

## Установка и настройка PostgreSQL

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
pg_lsclusters
Ver Cluster Port Status Owner    Data directory              Log file
15  main    5432 online postgres /var/lib/postgresql/15/main /var/log/postgresql/postgresql-15-main.log
```

Запускаем psql от пользователя postgres:

```
sudo -u postgres psql
```

Создаем таблицу, вносим данные и выходим из psql:

```
postgres=# create table test(c1 text);
CREATE TABLE
postgres=# insert into test values('1');
INSERT 0 1
postgres=# \q
```

Останавливаем postgresql и проверяем статус:

```
rsys@template:~$ sudo systemctl stop postgresql@15-main
rsys@template:~$ sudo systemctl status postgresql@15-main
○ postgresql@15-main.service - PostgreSQL Cluster 15-main
     Loaded: loaded (/usr/lib/systemd/system/postgresql@.service; enabled-runtime; preset: enabled)
     Active: inactive (dead) since Wed 2024-08-21 09:25:32 MSK; 1s ago
```

Добавляем к ВМ новый диск размером 10 Гб:

![](/assets/images/HW-3/1.png)

Проверяем, что диск виден системой:

```
rsys@template:~$ sudo fdisk -l|grep 10
Disk /dev/sdb: 10 GiB, 10737418240 bytes, 20971520 sectors
```

Создадим раздел на диске:

```
rsys@template:~$ sudo fdisk /dev/sdb

Command (m for help): n
Partition type
   p   primary (0 primary, 0 extended, 4 free)
   e   extended (container for logical partitions)
Select (default p): p
Partition number (1-4, default 1):
First sector (2048-20971519, default 2048):
Last sector, +/-sectors or +/-size{K,M,G,T,P} (2048-20971519, default 20971519):

Created a new partition 1 of type 'Linux' and of size 10 GiB.

Command (m for help): w
The partition table has been altered.
Calling ioctl() to re-read partition table.
Syncing disks.
```

Отформатируем созданный раздел /dev/sdb1 в ext4:

```
rsys@template:~$ sudo mkfs.ext4 /dev/sdb1
mke2fs 1.47.0 (5-Feb-2023)
Creating filesystem with 2621184 4k blocks and 655360 inodes
Filesystem UUID: 674de8ca-a88b-40f4-a0f4-67f6912c2985
Superblock backups stored on blocks:
	32768, 98304, 163840, 229376, 294912, 819200, 884736, 1605632

Allocating group tables: done
Writing inode tables: done
Creating journal (16384 blocks): done
Writing superblocks and filesystem accounting information: done
```

Узнаем UUID нашего раздела:

```
lsblk -f|grep sdb1
```

Создадим каталог, куда будем монтировать раздел:

```
sudo mkdir /mnt/data
```

Добавим запись о разделе в /etc/fstab:

```
rsys@template:~$ echo '/dev/disk/by-uuid/674de8ca-a88b-40f4-a0f4-67f6912c2985 /mnt/data ext4 defaults 0 1' | sudo tee -a /etc/fstab
/dev/disk/by-uuid/674de8ca-a88b-40f4-a0f4-67f6912c2985 /mnt/data ext4 defaults 0 1
```

Проверим, что раздел пока не смонтирован, затем проведем монтирование и проверим смонитровался ли наш раздел в каталог:

```
rsys@template:~$ sudo mount|grep sdb1
rsys@template:~$ sudo mount -a
rsys@template:~$ sudo mount|grep sdb1
/dev/sdb1 on /mnt/data type ext4 (rw,relatime)
```

Все прошло успешно, перезагружаемся и проверяем монтируется ли раздел после перезагрузки:

```
rsys@template:~$ sudo reboot
rsys@template:~$ sudo mount|grep sdb1
/dev/sdb1 on /mnt/data type ext4 (rw,relatime)
```

Делаем пользователя postgres владельцем данного каталога:

```
rsys@template:~$ sudo chown -R postgres:postgres /mnt/data/
```

Останавливаем postgresql и переносим данные в новый раздел:

```
rsys@template:~$ sudo systemctl stop postgresql@15-main.service
rsys@template:~$ sudo mv /var/lib/postgresql/15 /mnt/data/
```

Пробуем запустить кластер:

```
rsys@template:~$ sudo systemctl start postgresql@15-main.service
Job for postgresql@15-main.service failed because the service did not take the steps required by its unit configuration.
See "systemctl status postgresql@15-main.service" and "journalctl -xeu postgresql@15-main.service" for details.
```

Читаем журнал, убеждаемся, что причина в переносе каталога:

```
rsys@template:~$ sudo journalctl -xeu postgresql@15-main.service|grep Error
авг 21 09:53:28 template postgresql@15-main[1468]: Error: /var/lib/postgresql/15/main is not accessible or does not exist
```

Получаем все места конфигурационных файлов, где нам надо изменить запись и меняем /var/lib/postgresql/15/main на /mnt/data/15/main:

```
rsys@template:~$ sudo grep -r "/var/lib/postgresql/15/main" /etc/postgresql/*
/etc/postgresql/15/main/postgresql.conf:data_directory = '/var/lib/postgresql/15/main'		# use data in another directory
```

После изменения снова запускаем postgresql:

```
rsys@template:~$ sudo nano /etc/postgresql/15/main/postgresql.conf
rsys@template:~$ sudo systemctl start postgresql@15-main.service
rsys@template:~$ sudo systemctl status postgresql@15-main.service
● postgresql@15-main.service - PostgreSQL Cluster 15-main
     Loaded: loaded (/usr/lib/systemd/system/postgresql@.service; enabled-runtime; preset: enabled)
     Active: active (running) since Wed 2024-08-21 09:59:46 MSK; 7s ago
```

Подключаемся через psql и проверяем, что данные на месте:

```
rsys@template:~$ sudo -u postgres psql
postgres=# SELECT * FROM test;
 c1
----
 1
(1 row)
```

===================================================

## Задание со звездочкой:

Развернем еще 1 ВМ с аналогичными характерстиками:

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
rsys@template2:~$ pg_lsclusters
Ver Cluster Port Status Owner    Data directory              Log file
15  main    5432 online postgres /var/lib/postgresql/15/main /var/log/postgresql/postgresql-15-main.log
```

Останавливаем кластер и удаляем каталог с данными:

```
rsys@template2:~$ sudo systemctl stop postgresql@15-main.service
rsys@template2:~$ sudo rm -rf /var/lib/postgresql/
```

На **первой** ВМ останавливаем postgresql и размонтируем раздел:

```
rsys@template:~$ sudo systemctl stop postgresql@15-main.service
rsys@template:~$ sudo umount /mnt/data
```

Отключим диск с данными от первой ВМ и подключим ко второй ВМ:

![](/assets/images/HW-3/2.png)
![](/assets/images/HW-3/3.png)

Проверяем, что система увидела диск и раздел на нем:

```
rsys@template2:~$ sudo fdisk -l|grep 10
Disk /dev/sdb: 10 GiB, 10737418240 bytes, 20971520 sectors
/dev/sdb1        2048 20971519 20969472  10G 83 Linux
```

Создаем каталог /mnt/data, монтируем в него раздел присоединенного с ВМ 1 диска и даем на него права пользователю postgres:

```
rsys@template2:~$ sudo mkdir /mnt/data
rsys@template2:~$ sudo mount /dev/sdb1 /mnt/data/
rsys@template2:~$ sudo chown -R postgres:postgres /mnt/data/
```

Меняем параметр data_directory на "/mnt/data/15/main" в файле /etc/postgresql/15/main/postgresql.conf и запускаем postgresql:

```
rsys@template2:~$ sudo nano /etc/postgresql/15/main/postgresql.conf
rsys@template2:~$ sudo systemctl start postgresql@15-main.service
```

Подключаемся через psql и проверяем на месте ли наши данные после переноса диска с 1 ВМ на 2 ВМ:

```
rsys@template2:~$ sudo -u postgres psql
postgres=# SELECT * FROM test;
 c1
----
 1
(1 row)
```

