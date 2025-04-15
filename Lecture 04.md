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
- остановим PostgreSQL:
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
меняем на:
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


задание со звездочкой *: не удаляя существующий инстанс ВМ сделайте новый, поставьте на его PostgreSQL, удалите файлы с данными из /var/lib/postgres, перемонтируйте внешний диск который сделали ранее от первой виртуальной машины ко второй и запустите PostgreSQL на второй машине так чтобы он работал с данными на внешнем диске, расскажите как вы это сделали и что в итоге получилось.
