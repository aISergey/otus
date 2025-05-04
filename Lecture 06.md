#### Исходные параметры
- виртуальная машина:
> ОЗУ: 4GB
> Процессоров: 6
> Диск: 100GB
> ОС: Debian 12.2.0-14 (`cat /proc/version`)
> PostgreSQL: 17.4 (`/usr/lib/postgresql/17/bin/postgres -V`)

#### Настройка тестового кластера
- переходим к папке с конфигурацией: `cd /etc/postgresql/17/otus_1`
- открываем файл с конфигурацией: `sudo nano postgresql.conf`
- устанавливаем параметры:
```
shared_buffers = 1GB          # 25% от общего ОЗУ
max_connections = 200         # связать с work_mem
work_mem = 4MB                # 200 * 4 = 800 MB (что меньше, но близко к shared_buffers)
effective_cache_size = 3GB    # 75% от общего ОЗУ
wal_buffers = 16MB
```
- рестартуем кластер: `sudo pg_ctlcluster 17 otus_1 restart`
- переходим в папку с программой: `cd /usr/lib/postgresql/17/bin`
- инициализируем: `./pgbench -i -p 5433 testdb`
- запускаем тест: `./pgbench -p 5433 -c 200 -j 4 -P 30 -T 120 testdb`
> порт 5433 (кластер otus_1); 200 подключений; 4 потока; с выводом отчёта каждые 30 секунд; в течении 120 секунд; база данных testdb

**результат:**
```
number of transactions actually processed: 99359
number of failed transactions: 0 (0.000%)
latency average = 242.010 ms
latency stddev = 344.567 ms
initial connection time = 86.101 ms
tps = 824.337242 (without initial connection time)
```
- добавим к тесту -С: `./pgbench -p 5433 -c 200 -C -j 4 -P 30 -T 120 testdb`
> новое подключение для каждой транзакции

**результат:**
```
number of transactions actually processed: 64662
number of failed transactions: 0 (0.000%)
latency average = 370.441 ms
latency stddev = 640.979 ms
average connection time = 1.609 ms
tps = 535.861620 (including reconnection times)
```

#### Задание со звездой
- установим:
```
curl -s https://packagecloud.io/install/repositories/akopytov/sysbench/script.deb.sh | sudo bash
sudo apt -y install sysbench
```
- запустим: `sysbench --threads=4 --time=120 --report-interval=30 --db-driver=pgsql --pgsql-port=5433 --pgsql-user=postgres --pgsql-password=**** --pgsql-db=testdb cpu run`
> варианты: `fileio cpu memory threads mutex`

**результат:**
```
CPU speed:
    events per second: 14233.57

General statistics:
    total time:                          120.0012s
    total number of events:              1708057

Latency (ms):
         min:                                    0.26
         avg:                                    0.28
         max:                                    4.99
         95th percentile:                        0.29
         sum:                               479740.80

Threads fairness:
    events (avg/stddev):           427014.2500/125.28
    execution time (avg/stddev):   119.9352/0.00
```
