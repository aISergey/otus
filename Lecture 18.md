## Подготовка данных

- выполним скрипт, создадим и заполним исходные таблицы **goods, sales, good_sum_mart**

## Создадим триггер к sales

- создадим триггеры (для скорости работы, триггеры будут **for each statement**):
```
create or replace function pract_functions.tiud_sales()
returns trigger as $$
begin
  -- для простоты кода, полностью обновим изменяемые данные
  delete from pract_functions.good_sum_mart s
    using
      upd_data w inner join pract_functions.goods g on (w.good_id = g.goods_id)
    where
      -- join
      (s.good_name = g.good_name);
  --
  insert into pract_functions.good_sum_mart (good_name, sum_sale)
    select g.good_name, sum(g.good_price * s.sales_qty) as sum_sale
      from
        upd_data w
          inner join pract_functions.goods g on (w.good_id = g.goods_id)
          inner join sales s on (g.goods_id = s.good_id)
      group by g.good_name;
  --
  return null;
end; $$ language plpgsql;
--
create or replace trigger sales_ins
  after insert on pract_functions.sales
  referencing NEW table as upd_data
  for each statement execute function pract_functions.tiud_sales();
--
create or replace trigger sales_upd
  after update on pract_functions.sales
  referencing OLD table as old_data NEW table as upd_data
  for each statement execute function pract_functions.tiud_sales();
--
create or replace trigger sales_del
  after delete on pract_functions.sales
  referencing OLD table as upd_data
  for each statement execute function pract_functions.tiud_sales();
```
- контрольные скрипты:
```
select g.good_name, sum(g.good_price * s.sales_qty)
  from
    pract_functions.goods g
      inner join pract_functions.sales s on (g.goods_id = s.good_id)
  group by g.good_name;
--
select * from pract_functions.good_sum_mart;
```

- проверим работу триггеров:
```
insert into pract_functions.sales (good_id, sales_qty) VALUES (1, 40);
-- запустить контрольные скрипты
update sales set sales_qty = 60 where (sales_id = 7);
-- запустить контрольные скрипты
delete from sales where (sales_id = 7);
-- запустить контрольные скрипты
```

## Задание со звездой

- отторгаемые отчётные данные могут быть полезны для регистрации истории отчётности, что в принципе не возможно для запроса "по требованию", так как конструкция `goods g inner join sales s on (g.goods_id = s.good_id)` всегда будет брать текущее значение цен в goods.
