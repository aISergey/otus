#### Создадим новую БД (перед этим в template1 было добавлено расширение citext)
```
su postgres
psql
create database TEST;
\c TEST
create table test(c1 citext);
insert into test values('тест 1');
\q
```

#### Создадим новый диск /mnt/pg_data
( https://www.digitalocean.com/community/tutorials/how-to-partition-and-format-storage-devices-in-linux ) \
(ссылка доступна только через прокси)
#### Перенос данных кластера /17/main на новый диск
- остановим кластер:
> sudo systemctl stop postgresql@17-main
- дадим права пользователю postgres на новый раздел (станет владельцем):
> chown -R postgres:postgres /mnt/pg_data/
- переносим данные кластера:
> mv /var/lib/postgresql/17 /mnt/pg_data
- запускаем кластер:
> sudo systemctl start postgresql@17-main \
получаем ошибку - логично, данные кластера переехали!
- исправим ошибку, отредактируем файл /etc/postgresql/17/main/postgresql.conf
> data_directory = '/var/lib/postgresql/17/main' \
меняем на: \
> data_directory = '/mnt/pg_data/17/main'
- снова запускаем кластер:
> sudo systemctl start postgresql@17-main
- проверим, что всё ок:
```
su postgres
psql
\c test
select * from test;
```

#### Задание со звездой

- создаём новый инстанс виртуальной машины, установливаем и настраиваем PostgreSQL
- к новой машине примонтируем второй диск с данными первого инстанса
- создадим каталог в новой системе и примонтируем к нему новый диск на постоянной основе
- устанавливаем права postgres на примонтированный каталог
- остановим PostgreSQL
- удаляем каталог '/var/lib/postgresql/17/main'
- убеждаемся, что каталог '/var/lib/postgresql/17' пустой:
> ls /var/lib/postgresql/17
- редактируем файл '/etc/postgresql/17/main/postgresql.conf'
- запускаем кластер
- проверяем наличие базы данных TEST, таблицы test и её данных в виде кириллической строки, всё ок
- попытка запустить первый инстанс не проходит, т.к. файл второго диска уже занят первым инстансом

Задание выполнено!
