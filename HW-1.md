# Домашнее задание №1

## Работа с уровнями изоляции транзакции в PostgreSQL

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
sudo apt-get install postgresql-16
```

Проверяем, что всё прошло успешно и запущено:

```
pg_lsclusters
```
![](/assets/images/HW-1/1.png)

Запускаем psql от пользователя postgres:

```
sudo -u postgres psql
```

Проверяем, что стандартные таблицы созданы:

```
\l
```
![](/assets/images/HW-1/2.png)

Меняем пароль на пользователя postgres:

```
alter user postgres password 'sCkDIQz{?fxF#Acz';
```
> Пароль указан для примера.

Выключаем autocommit и проверяем, что он выключен:

```
\set AUTOCOMMIT off
\echo :AUTOCOMMIT
```
![](/assets/images/HW-1/3.png)


В **первой** ssh-сессии создаем таблицу и заполняем данными:

```
create table persons(id serial, first_name text, second_name text);
insert into persons(first_name, second_name) values('ivan', 'ivanov');
insert into persons(first_name, second_name) values('petr', 'petrov');
commit;
```
![](/assets/images/HW-1/4.png)

Проверяем текущий уровень изоляции:

```
SHOW TRANSACTION ISOLATION LEVEL;
```
![](/assets/images/HW-1/5.png)

Добавляем в **первой** сессии запись без коммита:

```
insert into persons(first_name, second_name) values('sergey', 'sergeev');
```
![](/assets/images/HW-1/6.png)

Во **второй** сесии выполняем запрос:

```
select * from persons;
```
![](/assets/images/HW-1/7.png)

> Новой записи (sergey sergeev) мы не видим, т.к. в **первой** сессии не был сделан commit, а уровень изоляции в данный момент read commited. 

Завершаем транзакцию в **первой** сессии:

```
commit;
```
![](/assets/images/HW-1/8.png)

Во **второй** сессии выполняем запрос:

```
select * from persons;
```
![](/assets/images/HW-1/9.png)

> Новую запись (sergey sergeev) мы видим, т.к. в **первой** сессии мы завершили транзакцию.

Завершаем транзакцию во второй сессии:

```
commit;
```

Меняем уровень изоляции в **обоих** сессиях и проверяем:

```
set transaction isolation level repeatable read;
SHOW TRANSACTION ISOLATION LEVEL;
```
![](/assets/images/HW-1/10.png)

В **первой** сессии добавляем новую запись без коммита:

```
insert into persons(first_name, second_name) values('sveta', 'svetova');
```
![](/assets/images/HW-1/11.png)

Во **второй** сессии выполняем запрос:

```
select * from persons;
```
![](/assets/images/HW-1/9.png)

> Новой записи (sveta svetova) нет.

Завершаем транзакцию в **первой** сессии:

```
commit;
```

Во **второй** сессии выполняем запрос:

```
select * from persons;
```
![](/assets/images/HW-1/9.png)

> Новой записи (sveta svetova) нет.

Завершаем транзакцию во **второй** сессии:

```
commit;
```

После завершения транзакции повторяем запрос во **второй** сессии:

```
select * from persons;
```
![](/assets/images/HW-1/12.png)

> Новая запись (sveta svetova) видна, т.к. предыдущая транзакция была завершена и выборка была получена заново.