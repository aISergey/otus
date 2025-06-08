## Тестирование индексов

- **тестовая таблица**:
```
create table test_index (col_id int, col_txt text, col_tsvector tsvector generated always as (col_txt::tsvector) stored);
insert into test_index (col_id, col_txt)
  select
      id as col_id,
      right('000' || (floor(random() * 1000) + 1)::text, 3) || ' ' || right('000' || (floor(random() * 1000) + 1)::text, 3) as col_txt
    from generate_series(1, 1000000) as id;
```

### Индекс на целочисленное поле

- **запрос**: `explain (analyze) select * from test_index where (col_id between 50000 and 250000);` \
**результат**:
```
Seq Scan on test_index  (cost=0.00..22353.00 rows=198012 width=30) (actual time=2.273..42.859 rows=200001 loops=1)
  Filter: ((col_id >= 50000) AND (col_id <= 250000))
  Rows Removed by Filter: 799999
Planning Time: 0.115 ms
Execution Time: 49.736 ms
```
**результат после создания индекса**: `create index on test_index(col_id); analyze test_index;`
```
Index Scan using test_index_col_id_idx on test_index  (cost=0.42..7582.74 rows=197666 width=30) (actual time=0.046..22.038 rows=200001 loops=1)
  Index Cond: ((col_id >= 50000) AND (col_id <= 250000))
Planning Time: 0.263 ms
Execution Time: 27.218 ms
```
**комментарий**: `скорость выросла в 1,5 раза, при этом доступ к первой записи упала практически до нуля: с 2.273 до 0.046`

### Полнотекстовый индекс

- **запрос**: `explain (analyze) select * from test_index where (col_tsvector @@ to_tsquery('306'));` \
**результат**:
```
Gather  (cost=1000.00..117898.00 rows=1700 width=30) (actual time=4.048..296.668 rows=2022 loops=1)
  Workers Planned: 2
  Workers Launched: 2
  ->  Parallel Seq Scan on test_index  (cost=0.00..116728.00 rows=708 width=30) (actual time=3.721..267.220 rows=674 loops=3)
        Filter: (col_tsvector @@ to_tsquery('306'::text))
        Rows Removed by Filter: 332659
Planning Time: 0.157 ms
JIT:
  Functions: 6
  Options: Inlining false, Optimization false, Expressions true, Deforming true
  Timing: Generation 0.775 ms (Deform 0.274 ms), Inlining 0.000 ms, Optimization 0.591 ms, Emission 8.265 ms, Total 9.631 ms
Execution Time: 297.044 ms
```
**результат после создания полнотекстового индекса**: `create index on test_index using gin(col_tsvector); analyze test_index;`
```
Bitmap Heap Scan on test_index  (cost=20.35..5324.30 rows=2200 width=30) (actual time=0.460..1.714 rows=2022 loops=1)
  Recheck Cond: (col_tsvector @@ to_tsquery('306'::text))
  Heap Blocks: exact=1766
  ->  Bitmap Index Scan on test_index_col_tsvector_idx  (cost=0.00..19.80 rows=2200 width=0) (actual time=0.282..0.282 rows=2022 loops=1)
        Index Cond: (col_tsvector @@ to_tsquery('306'::text))
Planning Time: 0.231 ms
Execution Time: 1.793 ms
```
**комментарий**: `прирост в скорости больше двух порядков! впечатляет...`

### Индекс на поле с функцией

- **запрос**: `explain (analyze) select * from test_index where (right(left(col_txt, 3), 1) || ' 1' = '6 1');` \
**результат**:
```
Gather  (cost=1000.00..17186.33 rows=5000 width=30) (actual time=0.199..78.159 rows=100284 loops=1)
  Workers Planned: 2
  Workers Launched: 2
  ->  Parallel Seq Scan on test_index  (cost=0.00..15686.33 rows=2083 width=30) (actual time=0.011..57.197 rows=33428 loops=3)
        Filter: (("right"("left"(col_txt, 3), 1) || ' 1'::text) = '6 1'::text)
        Rows Removed by Filter: 299905
Planning Time: 0.051 ms
Execution Time: 80.348 ms
```
**результат после создания простого индекса**: `create index on test_index(col_txt); analyze test_index;`
```
Gather  (cost=1000.00..17186.33 rows=5000 width=30) (actual time=0.214..84.950 rows=100284 loops=1)
  Workers Planned: 2
  Workers Launched: 2
  ->  Parallel Seq Scan on test_index  (cost=0.00..15686.33 rows=2083 width=30) (actual time=0.009..61.612 rows=33428 loops=3)
        Filter: (("right"("left"(col_txt, 3), 1) || ' 1'::text) = '6 1'::text)
        Rows Removed by Filter: 299905
Planning Time: 0.243 ms
Execution Time: 87.292 ms
```
**комментарий**: `время запроса даже увеличилось`
**результат после создания индекса с функцией на поле**: `create index on test_index((right(left(col_txt, 3), 1) || ' 1')); analyze test_index;`
```
Bitmap Heap Scan on test_index  (cost=1124.33..10489.99 rows=100633 width=30) (actual time=6.115..26.343 rows=100284 loops=1)
  Recheck Cond: (("right"("left"(col_txt, 3), 1) || ' 1'::text) = '6 1'::text)
  Heap Blocks: exact=7353
  ->  Bitmap Index Scan on test_index_expr_idx  (cost=0.00..1099.17 rows=100633 width=0) (actual time=5.111..5.112 rows=100284 loops=1)
        Index Cond: (("right"("left"(col_txt, 3), 1) || ' 1'::text) = '6 1'::text)
Planning Time: 0.325 ms
Execution Time: 29.822 ms
```
**комментарий**: `ну вот, теперь почти в 3 раза быстрее`

### Индекс на два поля

- **предварительно удалим все эти индексы**: `select * from pg_indexes where tablename = 'test_index';`
- **запрос**: `explain (analyze) select * from test_index where (col_id between 150000 and 650000) and (col_txt like '%6 1%');` \
**результат**:
```
Gather  (cost=1000.00..15649.67 rows=50 width=30) (actual time=9.239..47.165 rows=5003 loops=1)
  Workers Planned: 2
  Workers Launched: 2
  ->  Parallel Seq Scan on test_index  (cost=0.00..14644.67 rows=21 width=30) (actual time=3.022..25.745 rows=1668 loops=3)
        Filter: ((col_id >= 150000) AND (col_id <= 650000) AND (col_txt ~~ '%6 1%'::text))
        Rows Removed by Filter: 331666
Planning Time: 0.121 ms
Execution Time: 47.314 ms
```
**результат после создания индекса по двум полям**: `create index on test_index(col_id,col_txt); analyze test_index;`
```

```
**комментарий**: ` `
