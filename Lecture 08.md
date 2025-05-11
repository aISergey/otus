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

### 

<div class="text text_p-small text_default text_bold">Описание/Пошаговая инструкция выполнения домашнего задания:</div>

<div class="text text_p-small text_default learning-markdown js-learning-markdown"><ol>

<li>Сравните tps в синхронном/асинхронном режиме утилитой pgbench. Объясните полученный результат.</li>
<li>Создайте новый кластер с включенной контрольной суммой страниц. Создайте таблицу. Вставьте несколько значений. Выключите кластер. Измените пару байт в таблице. Включите кластер и сделайте выборку из таблицы. Что и почему произошло? как проигнорировать ошибку и продолжить работу? </li>
</ol>
</div>
