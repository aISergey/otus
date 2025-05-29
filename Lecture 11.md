## Структуры для тестов

- **products**:
```
create table products (product_id int, product_name text);
insert into products select id, 'Product ' || id::text from generate_series(1, 100) as id;
```
- **plans**:
```
create table plans (plan_id int, product_id int, plan_date date);
insert into plans select id, trunc(random() * 100 + 1), '2025-01-01'::date + trunc(random() * 365)::int from generate_series(1, 300) as id;
```
- **facts**:
```
create table facts (fact_id int, product_id int, fact_date date);
insert into facts select id, trunc(random() * 100 + 1), '2025-01-01'::date + trunc(random() * 365)::int from generate_series(1, 300) as id;
```

## Соединения таблиц

### inner join

- **plans**: `select count(*) from plans p inner join products s on (p.product_id = s.product_id);` \
**результат**: count=300 (плановые строки содержат только позиции из справочника номенклатуры)
- **facts**: `select count(*) from facts f inner join products s on (f.product_id = s.product_id);` \
**результат**: count=300 (строки факта содержат только позиции из справочника номенклатуры)
- **plans - facts**:
```
select count(*)
  from (
    select product_id, count(*) q from plans group by product_id
  ) p inner join (
    select product_id, count(*) q from facts group by product_id
  ) f on (p.product_id = f.product_id);
```
**результат**: count=87 (в плане и в факте встречается только 87 одинаковых позиций)

### left join

- **plans -> facts**:
```
select
    count(*) n_plan,
    sum(case when (f.product_id is not null) then 1 else 0 end) n_plan_fact,
    sum(case when (f.product_id is null) then 1 else 0 end) n_plan_only
  from (
    select product_id, count(*) q from plans group by product_id
  ) p left join (
    select product_id, count(*) q from facts group by product_id
  ) f on (p.product_id = f.product_id);
```
**результат**: n_plan=96, n_plan_fact=87 (уже видели в inner join), n_plan_only = 9

- **facts -> plans**:
```
select
    count(*) n_fact,
    sum(case when (p.product_id is not null) then 1 else 0 end) n_fact_plan,
    sum(case when (p.product_id is null) then 1 else 0 end) n_fact_only
  from (
    select product_id, count(*) q from facts group by product_id
  ) f left join (
    select product_id, count(*) q from plans group by product_id
  ) p on (f.product_id = p.product_id);
```
**результат**: n_fact=91, n_fact_plan=87 (уже видели в inner join), n_fact_only = 4

### cross join

- **plans, facts**:
```
select
    count(*)
  from (
    select product_id, count(*) q from plans group by product_id
  ) p cross join (
    select product_id, count(*) q from facts group by product_id
  ) f;
```
**результат**: count = 8736 (логично: декартово произведение n_plan=96 * n_fact=91)

### full join

- **plans <-> facts**:
```
select
    sum(case when (f.product_id is null) then 1 else 0 end) n_plan_only,
    sum(case when (f.product_id is not null) and (p.product_id is not null) then 1 else 0 end) n_plan_fact,
    sum(case when (p.product_id is null) then 1 else 0 end) n_fact_only
  from (
    select product_id, count(*) q from plans group by product_id
  ) p full join (
    select product_id, count(*) q from facts group by product_id
  ) f on (p.product_id = f.product_id);
```
**результат**:  n_plan_only=9, n_plan_fact=87, n_fact_only = 4

### multi join

- **plans - products -> facts**:

```
select
    p.product_id, s.product_name
  from (
    select product_id, count(*) q from plans group by product_id
  ) p inner join products s on (p.product_id = s.product_id)
      left join (
        select product_id, count(*) q from facts group by product_id
      ) f on (p.product_id = f.product_id)
  where
    (f.product_id is null);
```
**результат** (плановые 9 позиций, которых нет в факте):
```
 product_id | product_name
------------+--------------
          3 | Product 3
          5 | Product 5
          9 | Product 9
         10 | Product 10
         22 | Product 22
         31 | Product 31
         41 | Product 41
         44 | Product 44
         87 | Product 87
```

## Задание со звездой

- **общий для всех метрик скрипт**:
```
explain (analyze)
select p.*
  from (
    select product_id, count(*) q from plans group by product_id
  ) p inner join (
    select product_id, count(*) q from facts group by product_id
  ) f on (p.product_id = f.product_id);
```

- **nested loop**
```
set enable_nestloop=on;
set enable_hashjoin=off;
set enable_mergejoin=off;
```
**результат**:
```
Nested Loop  (cost=17.25..149.90 rows=91 width=12) (actual time=0.177..0.693 rows=87 loops=1)
...
Execution Time: 0.738 ms
```

- **hash join**
```
set enable_nestloop=off;
set enable_hashjoin=on;
set enable_mergejoin=off;
```
**результат** (эффективность по сравнению с "nested loop" выше в разы на текущих структурах!):
```
Hash Join  (cost=19.41..21.48 rows=91 width=12) (actual time=0.175..0.196 rows=87 loops=1)
...
Execution Time: 0.240 ms
```

- **merge join** (по сравнению с "hash join" видны дополнительные затраты на сортировку):
```
set enable_nestloop=off;
set enable_hashjoin=off;
set enable_mergejoin=on;
```
**результат**:
```
Hash Join  (cost=19.41..21.48 rows=91 width=12) (actual time=0.175..0.196 rows=87 loops=1)
...
Execution Time: 0.299 ms
```

- **а что выбирает оптимизатор?**:
```
set enable_nestloop=on;
set enable_hashjoin=on;
set enable_mergejoin=on;
```
**результат**:
```
Hash Join  (cost=19.41..21.48 rows=91 width=12) (actual time=0.188..0.210 rows=87 loops=1)
...
Execution Time: 0.245 ms
```
