## Работа с транзакциями в PostgreSQL
- открываем две сессии в psql
- в обеих отключаем AUTOCOMMIT:
```
\set AUTOCOMMIT off
```
- в первой сессии создаём таблицу и заполняем её данными:
```
create table persons(id serial, first_name text, second_name text);
insert into persons(first_name, second_name) values('ivan', 'ivanov'), ('petr', 'petrov');
commit;
```
- смотрим на текущий уровень изоляции:
```
show transaction isolation level;
```
ответ: read committed

## read committed транзакции

- в обоих терминалах открываем транзакции: begin;
- в первом терминале: insert into persons(first_name, second_name) values('sergey', 'sergeev');
- во втором терминале: select * from persons;
результат: две записи, ПОЧЕМУ?

- в первом терминале: commit;
- во втором терминале: select * from persons;
результат: три записи, ПОЧЕМУ?
- закрываем транзакцию во втором терминале

#### repeatable read транзации

- в обоих терминалах: begin; set transaction isolation level repeatable read;
- в первом терминале: insert into persons(first_name, second_name) values('sveta', 'svetova');
- во втором терминале: select * from persons;
результат: три записи, новой записи нет, ПОЧЕМУ?
- в первом терминале: commit;
- во втором терминале: select * from persons;
результат: три записи, новой записи нет, ПОЧЕМУ?
- во втором терминале: commit;
- во втором терминале: select * from persons;
результат: четыре записи, ПОЧЕМУ?
