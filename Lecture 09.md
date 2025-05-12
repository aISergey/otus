### Тест блокировок - данные журнала

> пересоздадим и настроим кластер
> создадим тестовую базу данных test_lock
> настроим у кластера регистрацию блокировок по таймауту:
```
alter system set log_lock_waits = on;
alter system set deadlock_timeout = '200ms';
select pg_reload_conf();
show deadlock_timeout;
```
> создадим тестовую таблицу:
```
create table tbl (id int primary key, data text);
insert into tbl values (1, 'str1'),(2, 'str2'),(3, 'str3');
```
**session 1**: `begin; update tbl set data = 'str1-1.1' where (id = 1);` \
**session 2**: `begin; update tbl set data = 'str1-2.1' where (id = 1);` --> ожидает сессию 1 \
**session 1**: `select pg_sleep(1); commit;` --> сессия 2 тоже обновила запись \
**session 2**: `commit;`
> смотрим журнал: `tail -n 10 /var/log/postgresql/postgresql-17-main.log`
```
2025-05-12 10:12:25.129 MSK [9646] postgres@test_lock СООБЩЕНИЕ:  процесс 9646 получил в режиме ShareLock блокировку "транзакция 755" через 7031.894 мс
2025-05-12 10:12:25.129 MSK [9646] postgres@test_lock КОНТЕКСТ:  при изменении кортежа (0,5) в отношении "tbl"
2025-05-12 10:12:25.129 MSK [9646] postgres@test_lock ОПЕРАТОР:  update tbl set data = 'str1-2.1' where (id = 1);
```

### Тест блокировок - представление pg_locks

- **session 1**: `begin; update tbl set data = 'str1-1.1' where (id = 1);`
- **session 2**: `begin; update tbl set data = 'str1-2.1' where (id = 1);`
- **session 3**: `begin; update tbl set data = 'str1-3.1' where (id = 1);`
- **session 4**: `select pid, locktype, relation::regclass, mode, granted, virtualtransaction, page, tuple from pg_locks where (locktype != 'virtualxid') and ((relation::regclass)::text != 'pg_locks') order by pid, locktype, (relation::regclass)::text;` \
**результат**:
```
 pid  | locktype | relation |       mode       | granted | virtualtransaction | page | tuple
------+----------+----------+------------------+---------+--------------------+------+-------
 9642 | relation | tbl      | RowExclusiveLock | t       | 5/10               |      |
 9642 | relation | tbl_pkey | RowExclusiveLock | t       | 5/10               |      |
 9646 | relation | tbl      | RowExclusiveLock | t       | 7/5                |      |
 9646 | relation | tbl_pkey | RowExclusiveLock | t       | 7/5                |      |
 9646 | tuple    | tbl      | ExclusiveLock    | t       | 7/5                |    0 |     7
 9837 | relation | tbl      | RowExclusiveLock | t       | 12/2               |      |
 9837 | relation | tbl_pkey | RowExclusiveLock | t       | 12/2               |      |
 9837 | tuple    | tbl      | ExclusiveLock    | f       | 12/2               |    0 |     7
```
**видим**:
  * блокировку строки у таблицы tbl тремя процессами
  * блокировку индекса первичного ключа тремя процессами
  * пропущены: блокировки на саму таблицу pg_locks и на собственные номера виртуальной транзакции
  * session 3 (9837) не получил достп к tbl \

- **session 1**: `commit;`
- новое состояние:
```
 pid  | locktype | relation |       mode       | granted | virtualtransaction | page | tuple
------+----------+----------+------------------+---------+--------------------+------+-------
 9646 | relation | tbl      | RowExclusiveLock | t       | 7/5                |      |
 9646 | relation | tbl_pkey | RowExclusiveLock | t       | 7/5                |      |
 9837 | relation | tbl      | RowExclusiveLock | t       | 12/2               |      |
 9837 | relation | tbl_pkey | RowExclusiveLock | t       | 12/2               |      |
```
**видим**:
  - session 3 (9837) получил достп к tbl
  - полностью пропал тип tuple: цепочки блокировок больше нет, и отпала необходимость в приоритизации доступа

### Тест блокировок - взаимоблокировка трех транзакций

- **session 1**: `begin; update tbl set data = 'str1-1.1' where (id = 1);`
- **session 2**: `begin; update tbl set data = 'str1-2.2' where (id = 2);`
- **session 3**: `begin; update tbl set data = 'str1-3.3' where (id = 3);`
- **session 1**: `update tbl set data = 'str1-1.2' where (id = 2);`
- **session 2**: `update tbl set data = 'str1-2.3' where (id = 3);`
```
 pid  | locktype | relation |       mode       | granted | virtualtransaction | page | tuple
------+----------+----------+------------------+---------+--------------------+------+-------
 9642 | relation | tbl      | RowExclusiveLock | t       | 5/20               |      |
 9642 | relation | tbl_pkey | RowExclusiveLock | t       | 5/20               |      |
 9642 | tuple    | tbl      | ExclusiveLock    | t       | 5/20               |    0 |     2
 9646 | relation | tbl      | RowExclusiveLock | t       | 7/6                |      |
 9646 | relation | tbl_pkey | RowExclusiveLock | t       | 7/6                |      |
 9646 | tuple    | tbl      | ExclusiveLock    | t       | 7/6                |    0 |     3
 9837 | relation | tbl      | RowExclusiveLock | t       | 12/3               |      |
 9837 | relation | tbl_pkey | RowExclusiveLock | t       | 12/3               |      |
```
- **session 3**: `update tbl set data = 'str1-3.1' where (id = 1);`
```
ОШИБКА:  обнаружена взаимоблокировка
DETAIL:  Процесс 9837 ожидает в режиме ShareLock блокировку "транзакция 760"; заблокирован процессом 9642.
Процесс 9642 ожидает в режиме ShareLock блокировку "транзакция 761"; заблокирован процессом 9646.
Процесс 9646 ожидает в режиме ShareLock блокировку "транзакция 762"; заблокирован процессом 9837.
CONTEXT:  при изменении кортежа (0,10) в отношении "tbl"
```
**состояние блокировок после ошибки**:
```
 pid  | locktype | relation |       mode       | granted | virtualtransaction | page | tuple
------+----------+----------+------------------+---------+--------------------+------+-------
 9642 | relation | tbl      | RowExclusiveLock | t       | 5/20               |      |
 9642 | relation | tbl_pkey | RowExclusiveLock | t       | 5/20               |      |
 9642 | tuple    | tbl      | ExclusiveLock    | t       | 5/20               |    0 |     2
 9646 | relation | tbl      | RowExclusiveLock | t       | 7/6                |      |
 9646 | relation | tbl_pkey | RowExclusiveLock | t       | 7/6                |      |
```
**session 1** ждёт коммита **session 2**

**данные в журнале**:
```
2025-05-12 11:52:58.921 MSK [9837] postgres@test_lock ОШИБКА:  обнаружена взаимоблокировка
2025-05-12 11:52:58.921 MSK [9837] postgres@test_lock ПОДРОБНОСТИ:  Процесс 9837 ожидает в режиме ShareLock блокировку "транзакция 760"; заблокирован процессом 9642.
        Процесс 9642 ожидает в режиме ShareLock блокировку "транзакция 761"; заблокирован процессом 9646.
        Процесс 9646 ожидает в режиме ShareLock блокировку "транзакция 762"; заблокирован процессом 9837.
        Процесс 9837: update tbl set data = 'str1-3.1' where (id = 1);
        Процесс 9642: update tbl set data = 'str1-1.2' where (id = 2);
        Процесс 9646: update tbl set data = 'str1-2.3' where (id = 3);
2025-05-12 11:52:58.921 MSK [9837] postgres@test_lock ПОДСКАЗКА:  Подробности запроса смотрите в протоколе сервера.
2025-05-12 11:52:58.921 MSK [9837] postgres@test_lock КОНТЕКСТ:  при изменении кортежа (0,10) в отношении "tbl"
2025-05-12 11:52:58.921 MSK [9837] postgres@test_lock ОПЕРАТОР:  update tbl set data = 'str1-3.1' where (id = 1);
```

### Задание со звездой

> вариант с предварительными select:
```
create table a (id int primary key, flag boolean);
insert into a values (1, 't'), (2, 'f');
```
**session 1**: `begin; select flag from a where (id = 1) for update;`

**session 2**: `begin; select flag from a where (id = 2) for update;`

> строки перепутаны, можно блокировать:

**session 1**: `update a set flag = 'f';` --> заблокированы второй сессией, ждём...

**session 2**: `update a set flag = 't';` --> заблокированы первой сессией, ошибка:
```
ОШИБКА:  обнаружена взаимоблокировка
DETAIL:  Процесс 8343 ожидает в режиме ShareLock блокировку "транзакция 740"; заблокирован процессом 8341.
Процесс 8341 ожидает в режиме ShareLock блокировку "транзакция 741"; заблокирован процессом 8343.
HINT:  Подробности запроса смотрите в протоколе сервера.
CONTEXT:  при изменении кортежа (0,1) в отношении "a"
```
