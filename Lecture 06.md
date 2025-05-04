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
- запускаем тест: `./pgbench -p 5433 -c 200 -j 4 -P 30 -T 120 testdb` \
> порт 5433 (кластер otus_1); 200 подключений; 4 потока; с выводом отчёта каждые 30 секунд; в течении 120 секунд; база данных testdb \
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

<div class="text text_p-small text_default learning-markdown js-learning-markdown"><ul>
<li>нагрузить кластер через утилиту через утилиту pgbench (<a target="_blank" href="https://postgrespro.ru/docs/postgrespro/14/pgbench" title="https://postgrespro.ru/docs/postgrespro/14/pgbench">https://postgrespro.ru/docs/postgrespro/14/pgbench</a>)</li>
<li>написать какого значения tps удалось достичь, показать какие параметры в какие значения устанавливали и почему</li>
</ul>
<p><br>Задание со *: аналогично протестировать через утилиту <a target="_blank" href="https://github.com/Percona-Lab/sysbench-tpcc" title="https://github.com/Percona-Lab/sysbench-tpcc">https://github.com/Percona-Lab/sysbench-tpcc</a> (требует установки<br><a target="_blank" href="https://github.com/akopytov/sysbench" title="https://github.com/akopytov/sysbench">https://github.com/akopytov/sysbench</a>)</p>
</div>
