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

- **запрос**: `explain (analyze) select * from test_index where (col_tsvector @@ to_tsquery('%306'));` \
**результат**:
```
Gather  (cost=1000.00..117898.00 rows=1700 width=30) (actual time=4.048..296.668 rows=2022 loops=1)
  Workers Planned: 2
  Workers Launched: 2
  ->  Parallel Seq Scan on test_index  (cost=0.00..116728.00 rows=708 width=30) (actual time=3.721..267.220 rows=674 loops=3)
        Filter: (col_tsvector @@ to_tsquery('%306'::text))
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
  Recheck Cond: (col_tsvector @@ to_tsquery('%306'::text))
  Heap Blocks: exact=1766
  ->  Bitmap Index Scan on test_index_col_tsvector_idx  (cost=0.00..19.80 rows=2200 width=0) (actual time=0.282..0.282 rows=2022 loops=1)
        Index Cond: (col_tsvector @@ to_tsquery('%306'::text))
Planning Time: 0.231 ms
Execution Time: 1.793 ms
```
**комментарий**: `прирост в скорости больше двух порядков! впечатляет...`

### Индекс на поле с функцией

- **запрос**: ` ` \
**результат**:
```

```
**результат после создания полнотекстового индекса**: ` `
```

```
**комментарий**: ` `

### Индекс на два поля

- **запрос**: ` ` \
**результат**:
```

```
**результат после создания полнотекстового индекса**: ` `
```

```
**комментарий**: ` `
