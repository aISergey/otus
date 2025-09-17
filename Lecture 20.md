## Логический бэкап COPY

- создаём тестовую таблицу и заполним её 100 записями (в ранее созданной БД **test_backup**): 
```
su postgres
psql
\c test_backup
create table test1(c text);
insert into test1(c) select md5(random()::text)::char(10) from generate_series(1,100);
select count(*) from test1;
\q
```
- создадим папку для бэкапов (под root):
```
cd /home
mkdir backup
chown -R postgres:postgres backup
```
- возвращаемся в сессию **postgres**, и делаем бэкап таблицы: `\copy test1 to '/home/backup/test1.sql';`
- посмотрим на файл в сессии **root**: `nano /home/backup/test1.sql`
- в сессии  **postgres** восстановим данные в новую таблицу:
```
create table test2(c text);
\copy test2 from '/home/backup/test1.sql';
select count(*) from test2;
```

## Логический бэкап PG_DUMP

- в предыдущем задании была база **test_backup** с двумя таблицами (**test1** и **test2**), \
  создадим архив базы **test_backup**:
```
pg_dump -d test_backup -Fc > /home/backup/test_backup.gz
ls -l /home/backup
```
- восстановим из backup в новую базу данных таблицу **test2**:
```
psql
create database test_backup_restore;
\q
pg_restore -d test_backup_restore -t test2 /home/backup/test_backup.gz
psql
\c test_backup_restore
\dt
select count(*) from test2;
```
