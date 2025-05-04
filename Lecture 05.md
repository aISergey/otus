#### Подготовка теста
- создадим новый кластер **otus_1**: ```sudo pg_createcluster --locale ru_RU.UTF-8 --start 17 otus_1```
- ещё раз проверим список работающих кластеров, видим рабочий порт нового кластера - 5433: ```sudo pg_lsclusters```
- настроим сеть для кластера:
>cd /etc/postgresql/17/otus_1 \
>sudo nano postgresql.conf: `listen_addresses = '*'` \
>sudo nano pg_hba.conf:
```
# Database administrative login by Unix domain socket
local   all             postgres                                peer
local   all             all                                     md5
# "local" is for Unix domain socket connections only
local   all             all                                     peer
# IPv4 local connections:
host    all             all             192.168.0.0/16          scram-sha-256
```
>sudo pg_ctlcluster 17 otus_1 restart \
>netstat -tnlpa | grep 5433 \
(видим: 0 0.0.0.0:5433)
- заходим в новый кластер:
```
su postgres
psql -p 5433
```

#### Основная работа

- создадим базу данных testdb:
```
create database testdb;
\c testdb
```
- создаём схему **testnm**: ```CREATE SCHEMA testnm;```
- новая таблица **testnm.t1**: ```CREATE TABLE testnm.t1(c1 integer);```
- добавим данные в **testnm.t1**: ```INSERT INTO testnm.t1 values(1);```
- новая роль **readonly**: ```CREATE role readonly;```
- права **readonly** на подключение к **testdb**: ```grant connect on DATABASE testdb TO readonly;```
- права **readonly** на использование схемы **testnm**: ```grant usage on SCHEMA testnm to readonly;```
- права **readonly** на SELECT всех таблиц схемы **testnm**: ```grant SELECT on all TABLEs in SCHEMA testnm TO readonly;```
- новый пользователь **testread**: ```CREATE USER testread with password 'test123';```
- добавим пользователя **testread** в роль **readonly**: ```grant readonly TO testread;```
- откроем новый терминал, и зайдём в кластер **otus_1** под пользователем **testread**:```psql -p 5433 -U testread -d testdb -W```
- прочитаем данные таблицы **testnm.t1**: ```select * from testnm.t1;``` \
**результат:**: всё хорошо, данные читаются (оказывается, в инструкции сознательно была допущена ошибка, которую я изначально исправил, пичаль... )))  )
- под пользователем **testread** пробуем создать таблицу: ```create table t2(c1 integer);``` \
**результат:**: ОШИБКА: нет доступа к схеме public \
для PostgreSQL 17 никаких прав на PUBLIC !!!
