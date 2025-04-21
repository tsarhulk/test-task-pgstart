# Задача 4
**Ускорить запрос "semi-join", добиться времени выполнения < 10sec**
```sql
select day from t2 where t_id in ( select t1.id from t1 where t2.t_id = t1.id) and day > to_char(date_trunc('day',now()- '1 months'::interval),'yyyymmdd');
```
## Решение
Наученные горьким опытом предыдущего задания, не будем запускать запрос, а просто посмотрим его план выполнения.
```sql
explain select day from t2 where t_id in ( select t1.id from t1 where t2.t_id = t1.id) and day > to_char(date_trunc('day',now()- '1 months'::interval),'yyyymmdd');
--                                                     QUERY PLAN
---------------------------------------------------------------------------------------------------------------------
-- Seq Scan on t2  (cost=0.00..520987139796.98 rows=412238 width=9)
--   Filter: ((day > to_char(date_trunc('day'::text, (now() - '1 mon'::interval)), 'yyyymmdd'::text)) AND (SubPlan 1))
--   SubPlan 1
--     ->  Seq Scan on t1  (cost=0.00..208398.00 rows=1 width=4)
--           Filter: (t2.t_id = id)
--(5 rows)
```
Ситуация повторяется. \
Благодаря оператору `in`, происходит дорогостоящее последовательное сканирование таблицы `t2`. Это связано с самим механизмом его действия: значения левого от `in` выражения сравниваются построчно со значениями во всех строках, возвращённых подзапросом.\
При использовании же `exists` подзапрос может выполняться не полностью, а завершаться, как только будет возвращена хотя бы одна строка.[[1]](https://postgrespro.ru/docs/postgresql/9.6/functions-subquery)

Именно эта особенность и позволит нам ускорить запрос.\
Перепишем запрос "semi-join", используя `exists`.
```sql
explain analyse select day from t2 where exists ( select 1 from t1 where t2.t_id = t1.id) and day > to_char(date_trunc('day', now() - '1 months'::interval), 'yyyymmdd');
--                                                         QUERY PLAN
-----------------------------------------------------------------------------------------------------------------------------
-- Hash Semi Join  (cost=347447.30..559886.24 rows=834347 width=9) (actual time=2381.540..9582.410 rows=812300 loops=1)
--   Hash Cond: (t2.t_id = t1.id)
--   ->  Seq Scan on t2  (cost=0.00..144370.27 rows=834347 width=13) (actual time=0.018..5287.235 rows=812300 loops=1)
--         Filter: (day > to_char(date_trunc('day'::text, (now() - '1 mon'::interval)), 'yyyymmdd'::text))
--         Rows Removed by Filter: 4187700
--   ->  Hash  (cost=183389.02..183389.02 rows=9999702 width=4) (actual time=2376.129..2376.130 rows=10000000 loops=1)
--         Buckets: 4194304  Batches: 8  Memory Usage: 76743kB
--         ->  Seq Scan on t1  (cost=0.00..183389.02 rows=9999702 width=4) (actual time=0.033..1058.040 rows=10000000 loops=1)
-- Planning Time: 2.279 ms
-- Execution Time: 9609.949 ms
--(10 rows)
```
Даже сам план запроса подсказывает нам, что мы сделали правильный выбор, ведь теперь там явно указано использование полусоединения.\
Время выполнения запроса меньше 10 секунд, а значит мы добились своего.

---
В порядке бреда перепишем первый запрос, используя внутреннее соединение таблиц.
```sql
explain analyze select day from t2 inner join t1 on t2.t_id = t1.id where t2.day > to_char(date_trunc('day', now() - '1 months'::interval), 'yyyymmdd');
--                                                        QUERY PLAN
---------------------------------------------------------------------------------------------------------------------------
-- Hash Join  (cost=154799.61..384030.98 rows=834347 width=9) (actual time=6229.114..9641.411 rows=812300 loops=1)
--   Hash Cond: (t1.id = t2.t_id)
--   ->  Seq Scan on t1  (cost=0.00..183389.02 rows=9999702 width=4) (actual time=0.043..916.455 rows=10000000 loops=1)
--   ->  Hash  (cost=144370.27..144370.27 rows=834347 width=13) (actual time=6226.499..6226.500 rows=812300 loops=1)
--         Buckets: 1048576  Batches: 1  Memory Usage: 46269kB
--         ->  Seq Scan on t2  (cost=0.00..144370.27 rows=834347 width=13) (actual time=0.088..6029.498 rows=812300 loops=1)
--               Filter: (day > to_char(date_trunc('day'::text, (now() - '1 mon'::interval)), 'yyyymmdd'::text))
--               Rows Removed by Filter: 4187700
-- Planning Time: 3.840 ms
-- Execution Time: 9666.235 ms
--(10 rows)
```
Время выполнения так же нас устраивает.\
Запросы выдают одинаковое количество строк.

Но в нашем случае таблица t1 используется только для проверки существования строки. Данные из неё мы не используем ни для установки фильтров и ограничений, ни в выводе.\
Корректнее будет использовать именно полусоединение, в котором возвращаются строки только одной из соединяемых таблиц. <s>А в случае с exists вообще выход бинарный, а не значения строки.</s>
