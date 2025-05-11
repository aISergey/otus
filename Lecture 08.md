### Checkpoints

- заново создадим кластер и сделаем его базовые настройки
- открываем **postgresql.conf**: `checkpoint_timeout = 30s`
- рестартуем кластер: `sudo pg_ctlcluster 17 main restart`
- создадим базу данных test_log и настроим pg_bench: `gbench -i test_log`
- текущая контрольная точка: `select pg_current_wal_insert_lsn();` \
**результат**: 0/25C79B8
- сбросим статистику по контрольным точкам: `select pg_stat_reset_shared ('checkpointer');`
- убедимся, что всё по нулям: `SELECT * FROM pg_stat_checkpointer \gx`
- запускаем нагрузку: `pgbench -c 50 -j 4 -P 30 -T 600 test_log`
- текущая контрольная точка: `select pg_current_wal_insert_lsn();`
**результат**: 0/1ADEE120
- размер журнальных записей: `select pg_size_pretty('0/1ADEE120'::pg_lsn - '0/25C79B8'::pg_lsn) as size;` \
**результат**: 392 MB (на 20 контрольных точек приходится в среднем 19.6 MB)
- статистика по контрольным точкам: `SELECT * FROM pg_stat_checkpointer \gx`
- **результат**:
```
num_timed           | 23
num_requested       | 0
restartpoints_timed | 0
restartpoints_req   | 0
restartpoints_done  | 0
write_time          | 564304
sync_time           | 1112
buffers_written     | 41792
stats_reset         | 2025-05-11 13:36:29.238308+03
```
все точки прошли только по расписанию (прошло чуть больше 20, т.к. задержался с запуском запроса)

### Тестирование режимов записи данных

- получим текущий режим записи: `show synchronous_commit;` => **результат**: synchronous_commit=on
- нагрузочное тестирование: `sudo -u postgres pgbench -P 1 -T 10 test_log` \
**результат**: tps = 967.352842 (without initial connection time)
- меняем режим: `alter system set synchronous_commit = off;`
- рестартуем: `sudo pg_ctlcluster 17 main reload`
- убеждаемся: `show synchronous_commit;` => **результат**: synchronous_commit=off
- повторяем тестирование: `sudo -u postgres pgbench -P 1 -T 10 test_log` \
**результат**: tps = 2642.101207 (without initial connection time) \
вау! скорость возросла почти в три раза. теперь мы не ждём подтверждения записи данных на диск, но, в случае сбоя, последние транзакции будут утеряны.

### Тестирование контрольных сумм страниц данных

- заново создадим кластер с контролем данных: ` sudo pg_createcluster --locale ru_RU.UTF-8 --start 17 main  -- --data-checksums`
- проверяем: `show data_checksums;` => **результат**: data_checksums=on
- создаём таблицу с данными:
```
create database test_data;
\c test_data
create table test_text(col1 text);
insert into test_text values ('str1'),('str2');
select * from test_text;
```
- получаем имя файла с данными: `select pg_relation_filepath('test_text');` \
**результат**: base/16388/16389
- выключаем кластер: `sudo pg_ctlcluster 17 main stop`
- открываем файл и "портим" его: `sudo dd if=/dev/zero of=/var/lib/postgresql/17/main/base/16388/16389 oflag=dsync conv=notrunc bs=1 count=8`
- включаем кластер: `sudo pg_ctlcluster 17 main start`
- пытаемся получить данные: `select * from test_text;` \
**результат**:
```
ПРЕДУПРЕЖДЕНИЕ:  ошибка проверки страницы: получена контрольная сумма 25342, а ожидалась - 31344
ОШИБКА:  неверная страница в блоке 0 отношения base/16388/16389
```
- устанавливаем параметр: `SET ignore_checksum_failure = on;`
- проверяем: `show ignore_checksum_failure;`
- пробуем прочитать данные: `select * from test_text;`
**результат**:
```
ПРЕДУПРЕЖДЕНИЕ:  ошибка проверки страницы: получена контрольная сумма 25342, а ожидалась - 31344
 col1
------
 str1
 str2
(2 rows)
```
- интересно, запрос залечивает таблицу: `update test_text set col1='str1' where (col1='str1');`
