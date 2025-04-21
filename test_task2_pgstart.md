# Задача 2
**Ускорить запрос "max + left join", добиться времени выполнения < 10ms**
```sql
select max(t2.day) from t2 left join t1 on t2.t_id = t1.id and t1.name like 'a%';
```
## Решение
> Спойлер.\
Возможно у меня <s>не хватило фантазии</s> оказалось недостаточно знаний и навыков для полноценного решения этой задачи, но это не мне решать.

Для начала посмотрим, как в целом работает наш запрос.
```sql
explain analyze select max(t2.day) from t2 left join t1 on t2.t_id = t1.id and t1.name like 'a%';
--                                                              QUERY PLAN
---------------------------------------------------------------------------------------------------------------------------------------
--  Aggregate  (cost=16547966.50..16547966.51 rows=1 width=32) (actual time=9055.276..9055.277 rows=1 loops=1)
--    ->  Merge Right Join  (cost=770962.78..14295430.83 rows=901014269 width=32) (actual time=4398.280..6750.582 rows=5000000 loops=1)
--          Merge Cond: (t1.id = t2.t_id)
--          ->  Sort  (cost=212385.39..212510.48 rows=50035 width=4) (actual time=1504.376..1532.279 rows=625548 loops=1)
--                Sort Key: t1.id
--                Sort Method: quicksort  Memory: 16384kB
--                ->  Seq Scan on t1  (cost=0.00..208480.00 rows=50035 width=4) (actual time=0.098..1467.686 rows=625548 loops=1)
--                      Filter: (name ~~ 'a%'::text)
--                      Rows Removed by Filter: 9374452
--          ->  Materialize  (cost=558577.39..576585.07 rows=3601536 width=36) (actual time=2893.899..4421.591 rows=5000000 loops=1)
--                ->  Sort  (cost=558577.39..567581.23 rows=3601536 width=36) (actual time=2893.885..3931.773 rows=5000000 loops=1)
--                      Sort Key: t2.t_id
--                      Sort Method: external merge  Disk: 127112kB
--                      ->  Seq Scan on t2  (cost=0.00..67887.36 rows=3601536 width=36) (actual time=0.060..935.689 rows=5000000 loops=1)
--  Planning Time: 2.009 ms
--  Execution Time: 9069.804 ms
-- (16 rows)
```
Честно говоря, не этого я ожидала. Мы пока не будем смотреть, что здесь используется именно метод объединения данных `Merge Right Join`. Нас интересует, что этому методу во время сортировки при материализации потребовалась дополнительная память на диске (то есть происходит внешняя сортировка).

Это вопрос решаемый, увеличим значение параметра `work_mem`.
```sql
alter system set work_mem = '64MB';
--ALTER SYSTEM
select pg_reload_conf();
-- pg_reload_conf
------------------
-- t
--(1 row)


explain analyze select max(t2.day) from t2 left join t1 on t2.t_id = t1.id and t1.name like 'a%';
--                                                            QUERY PLAN
--------------------------------------------------------------------------------------------------------------------------------
--  Aggregate  (cost=332112.01..332112.02 rows=1 width=32) (actual time=5876.301..5876.304 rows=1 loops=1)
--    ->  Hash Left Join  (cost=215963.81..319612.62 rows=4999756 width=9) (actual time=1606.803..3769.206 rows=5000000 loops=1)
--          Hash Cond: (t2.t_id = t1.id)
--          ->  Seq Scan on t2  (cost=0.00..81869.56 rows=4999756 width=13) (actual time=0.065..513.479 rows=5000000 loops=1)
--          ->  Hash  (cost=208388.28..208388.28 rows=606043 width=4) (actual time=1604.106..1604.108 rows=625548 loops=1)
--                Buckets: 1048576  Batches: 1  Memory Usage: 30184kB
--                ->  Seq Scan on t1  (cost=0.00..208388.28 rows=606043 width=4) (actual time=0.060..1451.661 rows=625548 loops=1)
--                      Filter: (name ~~ 'a%'::text)
--                      Rows Removed by Filter: 9374452
--  Planning Time: 1.170 ms
--  Execution Time: 5879.287 ms
-- (11 rows)
```
При увеличении памяти у нас поменялся метод объединения данных, и можно заметить, что данная комбинация эффективнее предыдущего варианта.

Также во время действия оператора `like` происходит последовательное сканирование (как и в прошлый запуск). Попробуем ускорить её, используя индекс.
```sql
create index t1_name on t1 (name text_pattern_ops);
--CREATE INDEX
analyze t1;
--ANALYZE
explain analyze select max(t2.day) from t2 left join t1 on t2.t_id = t1.id and t1.name like 'a%';
--                                                                     QUERY PLAN
----------------------------------------------------------------------------------------------------------------------------------------------------
--  Aggregate  (cost=235058.21..235058.22 rows=1 width=32) (actual time=4644.063..4644.066 rows=1 loops=1)
--    ->  Hash Left Join  (cost=118910.03..222558.82 rows=4999756 width=9) (actual time=852.346..2782.138 rows=5000000 loops=1)
--          Hash Cond: (t2.t_id = t1.id)
--          ->  Seq Scan on t2  (cost=0.00..81869.56 rows=4999756 width=13) (actual time=0.028..454.225 rows=5000000 loops=1)
--          ->  Hash  (cost=111335.76..111335.76 rows=605941 width=4) (actual time=850.727..850.729 rows=625548 loops=1)
--                Buckets: 1048576  Batches: 1  Memory Usage: 30184kB
--                ->  Bitmap Heap Scan on t1  (cost=20519.48..111335.76 rows=605941 width=4) (actual time=134.581..718.960 rows=625548 loops=1)
--                      Filter: (name ~~ 'a%'::text)
--                      Heap Blocks: exact=83300
--                      ->  Bitmap Index Scan on t1_name  (cost=0.00..20367.99 rows=593943 width=0) (actual time=120.164..120.164 rows=625548 loops=1)
--                            Index Cond: ((name ~>=~ 'a'::text) AND (name ~<~ 'b'::text))
--  Planning Time: 2.455 ms
--  Execution Time: 4647.760 ms
-- (13 rows)
```
Я использовала индекс с классом операторов `text_pattern_ops`, чтобы `like` смог проводить по нему фильтрацию.

Время выполнения улучшилось, но это оказался мой потолок. 

---
Далее пойдут немного беспорядочные попытки улучшить ситуацию.

Перепишем запрос, используя `not exists` вместо `left join`.
>Предупреждаю сразу, я не уверена, что запрос правильно написан и что нам вообще нужно *объединять таблицы*.

```sql
explain analyze select max(t2.day) from t2 where not exists (select *  from t1 where t1.id = t2.t_id and t1.name like 'a%');
--                                                                     QUERY PLAN
----------------------------------------------------------------------------------------------------------------------------------------------------
--  Aggregate  (cost=272954.10..272954.11 rows=1 width=32) (actual time=4736.985..4736.988 rows=1 loops=1)
--    ->  Hash Anti Join  (cost=118910.03..261212.25 rows=4696741 width=9) (actual time=912.409..2867.033 rows=4687280 loops=1)
--          Hash Cond: (t2.t_id = t1.id)
--          ->  Seq Scan on t2  (cost=0.00..81869.56 rows=4999756 width=13) (actual time=0.124..464.597 rows=5000000 loops=1)
--          ->  Hash  (cost=111335.76..111335.76 rows=605941 width=4) (actual time=910.833..910.834 rows=625548 loops=1)
--                Buckets: 1048576  Batches: 1  Memory Usage: 30184kB
--                ->  Bitmap Heap Scan on t1  (cost=20519.48..111335.76 rows=605941 width=4) (actual time=122.034..763.646 rows=625548 loops=1)
--                      Filter: (name ~~ 'a%'::text)
--                      Heap Blocks: exact=83300
--                      ->  Bitmap Index Scan on t1_name  (cost=0.00..20367.99 rows=593943 width=0) (actual time=107.121..107.121 rows=625548 loops=1)
--                            Index Cond: ((name ~>=~ 'a'::text) AND (name ~<~ 'b'::text))
--  Planning Time: 0.830 ms
--  Execution Time: 4740.439 ms
-- (13 rows)
```
Добавим индексы на поля, используемые для объединения таблиц.
```sql
create index t1_id on t1 (id); analyze t1;
--CREATE INDEX
--ANALYZE
create index t2_t_id on t2 (t_id); analyze t2;
--CREATE INDEX
--ANALYZE
explain analyze select max(t2.day) from t2 where not exists (select *  from t1 where t1.id = t2.t_id and t1.name like 'a%');
--                                                                    QUERY PLAN
--------------------------------------------------------------------------------------------------------------------------------------------------
-- Aggregate  (cost=272964.74..272964.75 rows=1 width=32) (actual time=4498.139..4498.141 rows=1 loops=1)
--   ->  Hash Anti Join  (cost=118916.60..261222.50 rows=4696897 width=9) (actual time=806.536..2741.933 rows=4687280 loops=1)
--         Hash Cond: (t2.t_id = t1.id)
--         ->  Seq Scan on t2  (cost=0.00..81871.23 rows=4999923 width=13) (actual time=0.022..467.305 rows=5000000 loops=1)
--         ->  Hash  (cost=111339.97..111339.97 rows=606130 width=4) (actual time=805.119..805.120 rows=625548 loops=1)
--               Buckets: 1048576  Batches: 1  Memory Usage: 30184kB
--               ->  Bitmap Heap Scan on t1  (cost=20521.37..111339.97 rows=606130 width=4) (actual time=113.060..674.829 rows=625548 loops=1)
--                     Filter: (name ~~ 'a%'::text)
--                     Heap Blocks: exact=83300
--                     ->  Bitmap Index Scan on t1_name  (cost=0.00..20369.84 rows=594128 width=0) (actual time=98.873..98.873 rows=625548 loops=1)
--                           Index Cond: ((name ~>=~ 'a'::text) AND (name ~<~ 'b'::text))
-- Planning Time: 2.907 ms
-- Execution Time: 4501.950 ms
--(13 rows)
```
\*звуки грусти*

Потом я увеличивала `effective_cache_size` до 8GB и  `work_mem` до 128 MB, что повлияло абсолютно никак. Делала индекс из поля `t2.day`, чтобы в целом ускорить агрегацию (ничего не поменялось).

И разочарованно решила, что объединение таблиц нам вообще *не нужно*, так как именно этот процесс и занимает всё время.
```sql
create index t2_day on t2 (day); analyze t2;
--CREATE INDEX
--ANALYZE
explain analyze select max(t2.day) from t2;
--                                                                    QUERY PLAN
--------------------------------------------------------------------------------------------------------------------------------------------------
-- Result  (cost=0.45..0.46 rows=1 width=32) (actual time=0.402..0.403 rows=1 loops=1)
--   InitPlan 1 (returns $0)
--     ->  Limit  (cost=0.43..0.45 rows=1 width=9) (actual time=0.399..0.400 rows=1 loops=1)
--           ->  Index Only Scan Backward using t2_day on t2  (cost=0.43..104782.02 rows=5000090 width=9) (actual time=0.398..0.398 rows=1 loops=1)
--                 Index Cond: (day IS NOT NULL)
--                 Heap Fetches: 0
-- Planning Time: 1.099 ms
-- Execution Time: 0.419 ms
--(8 rows)
```
Выполнение запроса со сканированием по индексу выполняется гораздо эффективнее, чем без него (`Planning Time: 1.572 ms, Execution Time: 3602.678 ms` при последовательном сканировании). К тому же, так как у нас используется функция `max()`, сканирование по индексу идёт по убыванию, то есть выбирается первое значение, которое и является самым большим.

