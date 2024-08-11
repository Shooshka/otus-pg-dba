# Домашнее задание №2

## Установка и настройка PostgreSQL в контейнере Docker

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

Добавляем репозиторий Docker и производим update:

```
sudo apt-get install ca-certificates curl
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo apt-get update
```

Устанавливаем Docker:

```
sudo apt-get install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

Проверяем, что всё работает:

```
sudo docker run hello-world
```

```
Unable to find image 'hello-world:latest' locally
latest: Pulling from library/hello-world
c1ec31eb5944: Pull complete
Digest: sha256:1408fec50309afee38f3535383f5b09419e6dc0925bc69891e79d84cc4cdcec6
Status: Downloaded newer image for hello-world:latest

Hello from Docker!
This message shows that your installation appears to be working correctly.
```

Создаем каталог /var/lib/postgres:

```
sudo mkdir /var/lib/postgres
```

Создаем сеть для контейнеров:

```
sudo docker network create pgnetwork
```

Ответом будет id сети:

```
rsys@template:~$ sudo docker network create pgnetwork
f2468ce606b15577be7c6544ac741f696fe62e2aeff380fbfee6c60979514a2f
```

Разворачиваем контейнер с postgresql 15, задаем имя, подключаем нашу созданную сеть, передаем пароль пользователя postgres через переменную, публикуем порт 5432 и монтируем созданный каталог внутрь контейнера:

```
sudo docker run --name pgserver --network pgnetwork -e POSTGRES_PASSWORD=postgres -d -p 5432:5432 -v /var/lib/postgres:/var/lib/postgresql/data postgres:15

Unable to find image 'postgres:15' locally
15: Pulling from library/postgres
efc2b5ad9eec: Pull complete
f948b1d87282: Pull complete
d163496f34e4: Pull complete
4fc2b8654b2c: Pull complete
8cda900c1bc3: Pull complete
a9c916151f1f: Pull complete
03f481324fa2: Pull complete
5565fcca00f8: Pull complete
9793cbf5696f: Pull complete
3774108fd045: Pull complete
f49ff5e231b7: Pull complete
33710b2e93bf: Pull complete
dec91f352a34: Pull complete
2dfcdb0a11a5: Pull complete
Digest: sha256:99981723cfb0b44e09a7bac386eafde3c151bf427910d953e61d0f0ed39f596b
Status: Downloaded newer image for postgres:15
6fc4522ac6ba54059393d6e92e7e870f242e77ae1a3e77478bf5a5732cea5aef
```

Разворачиваем еще один контейнер, который будем использовать в качестве клиента, подключаем ему созданную ранее сеть, задаём имя и запускаем в нем psql  с параметрами подключения, используя внутреннее имя предыдущего контейнера:

```
sudo docker run -it --rm --network pgnetwork --name pgclient postgres:15 psql -h pgserver -U postgres


Password for user postgres:
psql (15.8 (Debian 15.8-1.pgdg120+1))
Type "help" for help.

postgres=#
```

Создаем таблицу и добавляем пару строк в неё, проверяем, что все добавлено и выходим:

```
postgres=# CREATE TABLE testtable (name text, surname text);
CREATE TABLE
postgres=# INSERT INTO testtable (name, surname) VALUES ('ivan', 'ivanov');
INSERT 0 1
postgres=# INSERT INTO testtable (name, surname) VALUES ('petr', 'petrov');
INSERT 0 1
postgres=# SELECT * from testtable;
 name | surname
------+---------
 ivan | ivanov
 petr | petrov
(2 rows)
postgres=# \q
```

На локальном ноутбуке с MacOS устанавливаем psql и добавляем PATH:

```
brew install libpq
echo 'export PATH="/opt/homebrew/opt/libpq/bin:$PATH"' >> ~/.zshrc
```

Смотрим id текущего контейнера с postgresql:

```
rsys@template:~$ sudo docker ps
CONTAINER ID   IMAGE         COMMAND                  CREATED          STATUS          PORTS                                       NAMES
6fc4522ac6ba   postgres:15   "docker-entrypoint.s…"   17 minutes ago   Up 17 minutes   0.0.0.0:5432->5432/tcp, :::5432->5432/tcp   pgserver
```

Запускаем bash внутри этого контейнера:

```
rsys@template:~$ sudo docker exec -it 6fc4522ac6ba bash
root@6fc4522ac6ba:/#
```

Устанавливаем редактор nano внутри контейнера:

```
apt-get update && apt-get install nano -y
```

Проверяем, что в postgresql.conf и в pg_hba.conf есть разрешения на подключение. В случае отсутствия меняем listen_address на = '*' в /var/lib/postgresql/data/postgresql.conf и добавляем строчку host all all all scram-sha-256 в /var/lib/postgresql/data/pg_hba.conf.

Подключаемся с локального ПК к PostgreSQL в контейнере на виртуальной машине:

```
shoo@MacBook-Air-Vyacheslav ~ % psql -h <Внешний IP нашей ВМ с Docker> -p 5432 -U postgres -W
Password: 
psql (16.4, server 15.8 (Debian 15.8-1.pgdg120+1))
Type "help" for help.

postgres=# SELECT * from testtable;
 name | surname 
------+---------
 ivan | ivanov
 petr | petrov
(2 rows) 
```

Удаляем контейнер с PostgreSQL:

```
rsys@template:~$ sudo docker ps
[sudo] password for rsys:
CONTAINER ID   IMAGE         COMMAND                  CREATED          STATUS          PORTS                                       NAMES
6fc4522ac6ba   postgres:15   "docker-entrypoint.s…"   31 minutes ago   Up 31 minutes   0.0.0.0:5432->5432/tcp, :::5432->5432/tcp   pgserver
rsys@template:~$ sudo docker stop 6fc4522ac6ba
6fc4522ac6ba
rsys@template:~$ sudo docker rm 6fc4522ac6ba
6fc4522ac6ba
rsys@template:~$ sudo docker ps
CONTAINER ID   IMAGE     COMMAND   CREATED   STATUS    PORTS     NAMES
rsys@template:~$
```

Создаем контейнер заново с такими же параметрами:

```
sudo docker run --name pgserver --network pgnetwork -e POSTGRES_PASSWORD=postgres -d -p 5432:5432 -v /var/lib/postgres:/var/lib/postgresql/data postgres:15

rsys@template:~$ sudo docker ps
CONTAINER ID   IMAGE         COMMAND                  CREATED          STATUS          PORTS                                       NAMES
86b203062efa   postgres:15   "docker-entrypoint.s…"   14 seconds ago   Up 14 seconds   0.0.0.0:5432->5432/tcp, :::5432->5432/tcp   pgserver
```

Снова создаем контейнер с клиентом и подключаемся к серверу:

```
sudo docker run -it --rm --network pgnetwork --name pgclient postgres:15 psql -h pgserver -U postgres

Password for user postgres:
psql (15.8 (Debian 15.8-1.pgdg120+1))
Type "help" for help.
```

Проверяем, что данные на месте:

```
postgres=# SELECT * from testtable;
 name | surname
------+---------
 ivan | ivanov
 petr | petrov
(2 rows)
```