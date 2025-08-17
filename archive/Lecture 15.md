## Подготовка данных

- загрузил базу **demo-medium**
- размеры таблиц: `boarding_passes=263М, ticket_flights=245М, tickets=133М, bookings=42М, ...`

## Секционирование таблицы boarding_passes

- попробуем разбить на секции **boarding_passes** (как самую большую) по посадочным номерам:
```
select right(s.seat_no, 1), count(*) n
  from bookings.boarding_passes s
  group by 1
  order by count(*) desc;
```
- всего 10 буквенных рядов, но лишь 6 диапазонов по примерному равенству количеств:
```
A	359512
D	339715
C	315815
F	273241
E	234615
B,G,H,K,J 371397
```
- создаём секционированную таблицу по списку:
```
create table bookings.boarding_passes_partitioned (
  ticket_no bpchar(13) not null,
  flight_id int4 not null,
  boarding_no int4 not null,
  seat_no varchar(4) not null
)
partition by list (right(seat_no, 1));
--
create table boarding_passes_partitioned_A partition of boarding_passes_partitioned for values in ('A');
create table boarding_passes_partitioned_D partition of boarding_passes_partitioned for values in ('D');
create table boarding_passes_partitioned_C partition of boarding_passes_partitioned for values in ('C');
create table boarding_passes_partitioned_F partition of boarding_passes_partitioned for values in ('F');
create table boarding_passes_partitioned_E partition of boarding_passes_partitioned for values in ('E');
create table boarding_passes_partitioned_def partition of boarding_passes_partitioned default;
```
- перенесём данные: `insert into bookings.boarding_passes_partitioned select * from bookings.boarding_passes;`
- проверим: `select right(s.seat_no, 1), count(*) n from bookings.boarding_passes_partitioned_A s group by 1;`
**результат**: A	359512 (вау!)

### Анализ результата

- исходная таблица: `explain (analyze) select * from bookings.boarding_passes s where (right(s.seat_no, 1) = 'A');`
```
Gather  (cost=1000.00..27738.44 rows=9471 width=25) (actual time=0.267..105.420 rows=359512 loops=1)
  Workers Planned: 2
  Workers Launched: 2
  ->  Parallel Seq Scan on boarding_passes s  (cost=0.00..25791.34 rows=3946 width=25) (actual time=0.027..91.361 rows=119837 loops=3)
        Filter: ("right"((seat_no)::text, 1) = 'A'::text)
        Rows Removed by Filter: 511594
Execution Time: 113.744 ms
```
- секционированная таблица: `explain (analyze) select * from bookings.boarding_passes_partitioned p where (right(p.seat_no, 1) = 'A');`
```
Gather  (cost=1000.00..6995.96 rows=1798 width=25) (actual time=0.204..40.618 rows=359512 loops=1)
  Workers Planned: 1
  Workers Launched: 1
  ->  Parallel Seq Scan on boarding_passes_partitioned_a p  (cost=0.00..5816.16 rows=1058 width=25) (actual time=0.009..24.275 rows=179756 loops=2)
        Filter: ("right"((seat_no)::text, 1) = 'A'::text)
Execution Time: 49.855 ms
```
**результат**: прирост скорости в 2.5 раза, впечатляет!

### Добавим ограничения

- восстановим FK:
```
ALTER TABLE bookings.boarding_passes_partitioned
  ADD CONSTRAINT boarding_passes_partitioned_ticket_no_fkey
    FOREIGN KEY (ticket_no, flight_id)
    REFERENCES bookings.ticket_flights(ticket_no,flight_id);
```
**результат**: скрипт прошёл без проблем, предыдущий запрос на скорость вернул тот же результат.

- попытка восстановить уникальные ключи потерпела крах - "Ограничения UNIQUE не могут использоваться, когда ключи секционирования включают выражения". Пример:
```
ALTER TABLE bookings.boarding_passes_partitioned
  ADD CONSTRAINT boarding_passes_partitioned_flight_id_boarding_no_key
  UNIQUE (flight_id, boarding_no);
```

## Общий вывод

Для секционирования таблицы существенным ограничением является наличие уникальных ключей. Поэтому секционировать желательно по "естественному ключу" - полю (или полям), а не выражению.
