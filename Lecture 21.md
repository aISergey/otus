## Логическая репликация

#### ВМ1
- настроим кластер
> sudo nano /etc/postgresql/17/main/postgresql.conf
```
wal_level = logical
wal_log_hints = on
```
> sudo nano /etc/postgresql/17/main/pg_hba.conf
```
host   replication   all   192.168.0.0/16   scram-sha-256
```
> рестартуем кластер:
```
sudo pg_ctlcluster 17 main restart
```
- настроим сервер баз данных
> создадим логин
```
su postgres
psql
create role usr_repl with
  LOGIN NOSUPERUSER NOCREATEDB NOCREATEROLE INHERIT REPLICATION NOBYPASSRLS CONNECTION LIMIT -1 PASSWORD 'usr';
```
> настроим структуру для публикации
```
create database db1;
\c db1
create table test1(col_int int primary key, col_str text);
grant select on table public.test1 to usr_repl;
alter table if exists public.test1 replica identity full;
create publication pub_db1_test1
  for table public.test1
    with (publish = 'insert, update, delete, truncate', publish_via_partition_root = false);
insert into test1(col_int, col_str) values (11, 'test 11');
```
#### ВМ2
- настройка кластера аналогична **ВМ1** (т.к. тоже будем публиковать данные)
- настройка сервера баз данных аналогична **ВМ1** (пароль пусть будет usr2)
- настроим структуру для подписки, и учтём текст из документации:
> https://www.postgresql.org/docs/current/logical-replication-subscription.html \
> **Replication to differently-named tables on the subscriber is not supported.** \
> т.е. придётся создавать одноимённую таблицу...
```
create database db2;
\c db2
create table test1(col_int int primary key, col_str text);
create subscription sub_srv1_db1_test1
  connection 'host=192.168.0.17 port=5432 user=usr_repl password=usr dbname=db1 connect_timeout=10 sslmode=prefer'
  publication pub_db1_test1 with
    (connect = true, enabled = true, copy_data = true, create_slot = true, synchronous_commit = 'off',
     binary = false, streaming = 'False', two_phase = false, disable_on_error = false,
     run_as_owner = false, password_required = true, origin = 'any');
select * from test1; -- должна появиться строка с данными!!!
```
интересно: PgAdmin, при формировании SQL-скрипта, теряет параметр password !!!
> создадим публикацию для **ВМ1**:
```
create table test2(id int primary key, nm text);
...
```
    
    <ol>
      <li>На 1 ВМ создаем таблицы test для записи, test2 для запросов на чтение. </li>
      <li>Создаем публикацию таблицы test и подписываемся на публикацию таблицы test2 с ВМ №2. </li>
      <li>На 2 ВМ создаем таблицы test2 для записи, test для запросов на чтение. </li>
      <li>Создаем публикацию таблицы test2 и подписываемся на публикацию таблицы test1 с ВМ №1. </li>
      <li>3 ВМ использовать как реплику для чтения и бэкапов (подписаться на таблицы из ВМ №1 и №2 ). </li>
    </ol>

    <p><br>* реализовать горячее реплицирование для высокой доступности на 4ВМ. Источником должна выступать ВМ №3. Написать с какими проблемами столкнулись.</p>
