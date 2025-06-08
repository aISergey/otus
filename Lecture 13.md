## Тестирование индексов

- **тестовая таблица**:
```
create table test_index (col_id int, col_txt text);
insert into test_index select id as col_id, 'idx_' || right('000' || (floor(random() * 1000) + 1)::text, 3) as col_txt
  from generate_series(1, 1000000) as id;
```

### Индекс на целочисленное поле

- **запрос**: `explain (analyze) select * from test_index where (col_id between 50000 and 250000);` \
**результат**:
```
Seq Scan on test_index  (cost=0.00..20406.00 rows=197036 width=12) (actual time=2.541..46.782 rows=200001 loops=1)
  Filter: ((col_id >= 50000) AND (col_id <= 250000))
  Rows Removed by Filter: 799999
Planning Time: 0.064 ms
Execution Time: 53.901 ms
```
**результат после создания индекса**: `create index on test_index(col_id); analyze test_index;`
```
Index Scan using test_index_col_id_idx on test_index  (cost=0.42..7228.24 rows=198541 width=12) (actual time=0.011..28.713 rows=200001 loops=1)
  Index Cond: ((col_id >= 50000) AND (col_id <= 250000))
Planning Time: 0.098 ms
Execution Time: 35.727 ms
```
**комментарий**: `скорость выросла в 1,5, при этом доступ к первой записи упала практически до нуля: с 2.541 до 0.011`

### Индекс на строковое поле

- **запрос**: ` ` \
**результат**:
```

```
**результат после создания полнотекстового индекса**: ` `
```

```
**комментарий**: ` `

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
