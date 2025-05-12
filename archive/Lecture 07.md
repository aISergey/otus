### Подготовка
- установлены параметры виртуальной машины: ОЗУ: 4GB; процессоров: 2
- создадим новый кластер: `sudo pg_createcluster --locale ru_RU.UTF-8 --start 17 otus_2`
- заходим под postgres: `su postgres`
- инициализируем тесты в базе postgres: `pgbench -p 5434 -i postgres`
- запускаем тест: `pgbench -p 5434 -c 8 -P 6 -T 60 postgres`

**результат:**
```
latency average = 6.862 ms
latency stddev = 5.438 ms
initial connection time = 12.861 ms
tps = 1165.361884 (without initial connection time)
```

### Установим параметры из задания
- переходим к папке с конфигурацией: `cd /etc/postgresql/17/otus_2`
- открываем файл с конфигурацией: `sudo nano postgresql.conf`
- установим параметры из задания:
```
max_connections = 40               # было: 100
shared_buffers = 1GB               # было: 128MB
effective_cache_size = 3GB         # было: 4GB
maintenance_work_mem = 512MB       # было: 64MB
checkpoint_completion_target = 0.9 # было: 0.9
wal_buffers = 16MB                 # было: -1 (1/32 от shared_buffers, т.е. 4MB)
default_statistics_target = 500    # было: 100
random_page_cost = 4               # было: 4.0
effective_io_concurrency = 2       # было: 1
work_mem = 6553kB                  # было: 4MB
min_wal_size = 4GB                 # было: 80MB
max_wal_size = 16GB                # было: 1GB
```
- рестартуем кластер: `sudo pg_ctlcluster 17 otus_2 restart`
- повторяем тест: `pgbench -p 5434 -c 8 -P 6 -T 60 postgres`

**результат:**
```
latency average = 7.152 ms
latency stddev = 7.019 ms
initial connection time = 9.977 ms
tps = 1118.300117 (without initial connection time)
```
гм. я бы сказал, что результат остался на прежнем уровне...

### Работа с VACUUM
- зайдём в кластер **otus_2**: `psql -p 5434`
- создадим таблицу и заполним её 1 млн. строк:
```
create table test_tbl(col1 text);
insert into test_tbl(col1) select generate_series::text from generate_series(1,1000000);
select pg_size_pretty(pg_total_relation_size('test_tbl'));
```
**результат:** 35 MB

- отключим autovacuum: `alter table test_tbl set (autovacuum_enabled = off);`
- обновим 5 раз таблицу, добавив символ 'a': `update test_tbl set col1 = col1 || 'a';`
- проверяем размер: 237MB
- количество мёртвых строк:
```
select relname, n_live_tup, n_dead_tup, trunc(100*n_dead_tup/(n_live_tup+1))::float "ratio%", last_autovacuum 
from pg_stat_user_TABLEs WHERE (relname = 'test_tbl');
```
**результат:**  n_dead_tup=4999870; ratio%=499; last_autovacuum=<5 минут назад>;  n_live_tup=1_000_000

- включаем autovacuum: `alter table test_tbl set (autovacuum_enabled = on);`
- подождали минуту: n_dead_tup=0; ratio%=0; last_autovacuum=<только что>;  n_live_tup=1_000_000
- размер: 237MB
- обновим таблицу ещё 5 раз, добавив символ 'b': `update test_tbl set col1 = col1 || 'b';`
- размер: 260MB
- через минуту мёртвые строки: n_dead_tup=0; ratio%=0; last_autovacuum=<только что>;  n_live_tup=1_649_168

### Задание со звездой

- анонимная процедура с циклом обновления всех строк в таблице с выводом информации о шаге цикла:
```
do $$
begin
  for i in 1..10 loop
    raise notice 'step: %', i;
    update test_tbl set col1 = col1 || i::text;
  end loop;
end $$ language plpgsql;
```
- размер: 569MB
- мёртвые строки: n_dead_tup=0; ratio%=0; last_autovacuum=<только что>;  n_live_tup=1_000_000
- проверим первую строчку: `select col1 from test_tbl where (col1 like '1a%');` \
**результат:** 1aaaaabbbbb12345678910
