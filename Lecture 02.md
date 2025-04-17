## Работа с транзакциями в PostgreSQL
- открываем две сессии в psql
- в обеих отключаем AUTOCOMMIT: ` \set AUTOCOMMIT off`
- в первой сессии создаём таблицу и заполняем её данными:
```
create table persons(id serial, first_name text, second_name text);
insert into persons(first_name, second_name) values('ivan', 'ivanov'), ('petr', 'petrov');
commit;
```
- смотрим на текущий уровень изоляции: ` show transaction isolation level; ` \
**ответ:** read committed

## read committed транзакции

- в обоих терминалах открываем транзакции:  `begin;`
- в первом терминале: `insert into persons(first_name, second_name) values('sergey', 'sergeev');`
- во втором терминале: `select * from persons;` \
**результат:** две записи, т.к. терминалы работают со своими отдельными копиями данных в рамках своих транзакций.

- в первом терминале: `commit;`
- во втором терминале: `select * from persons;` \
**результат:** три записи - первая сессия зафиксировала данные, а вторая сессия, при повторном чтении данных, обновляет данные своей копии (режим **nonrepeatable read**). \
из документации: `Also note that two successive SELECT commands can see different data, even though they are within a single transaction, if other transactions commit changes after the first SELECT starts and before the second SELECT starts.` \
и чуть ниже из документации: `Because Read Committed mode starts each command with a new snapshot that includes all transactions committed up to that instant, subsequent commands in the same transaction will see the effects of the committed concurrent transaction in any case. The point at issue above is whether or not a single command sees an absolutely consistent view of the database.`
- закрываем транзакцию во втором терминале

#### repeatable read транзации

- в обоих терминалах: `begin set transaction isolation level repeatable read;`
- в первом терминале: `insert into persons(first_name, second_name) values('sveta', 'svetova');`
- во втором терминале: `select * from persons;` \
**результат:** три записи, новой записи нет, т.к. терминалы работают со своими отдельными копиями данных в рамках своих транзакций.
- в первом терминале: `commit;`
- во втором терминале: `select * from persons;` \
**результат:** три записи, новой записи нет - первая сессия зафиксировала данные, но вторая сессия продолжает работать со своей копией данных, т.к. свою транзакцию ещё не закрыла (в PostgreSQL это уровень **snapshot isolation**). \
из документации: `The Repeatable Read isolation level only sees data committed before the transaction began; it never sees either uncommitted data or changes committed by concurrent transactions during the transaction's execution.` \
и чуть ниже из документации: `This level is different from Read Committed in that a query in a repeatable read transaction sees a snapshot as of the start of the first non-transaction-control statement in the transaction, not as of the start of the current statement within the transaction. Thus, successive SELECT commands within a single transaction see the same data, i.e., they do not see changes made by other transactions that committed after their own transaction started.`
- во втором терминале: `commit;`
- во втором терминале: `select * from persons;` \
**результат:** четыре записи - вторая сессия наконец-то закрыла свою транзакцию, и открыла новую, получив обновлённую копию актуальных данных.
