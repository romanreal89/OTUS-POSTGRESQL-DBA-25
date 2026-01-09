## Секционирование
Для выполнения ДЗ была загружена БД DEMO, с сайта postgrepro.  
### Выбор таблицы
Для секционирования подходит 3 таблицы с наибольшим количеством строк:
```
demo=# select count(*) from bookings.tickets;
  count
---------
 2949857
(1 row)

demo=# select count(*) from bookings.bookings;
  count
---------
 2111110
(1 row)
demo=# select count(*) from bookings.boarding_passes;
  count
---------
 7925812
(1 row)
```
Для секционирования была выбрана таблица bookings потому, что в этой таблице есть привязка к датам по полю book_date.   
В таблице flights тоже ест поля с датами, но сложнее выбрать колонку секционирования, 
поскольку запросы по данным столбцам могут пересекаться и не всегда запрос может быть оптимальным,  
а секционирование по рейсам или аэропортам будет скорее всего не оптимальным из-за их большого количества.   
Таблица boarding_passes тоже не имеет ярко выраженного ключа для секционирования, 
и ее можно секционировать разве что по хэшу, но это не даст выигрыша в производительности при поиске данных.   
В остальных таблицах данных не так много, чтобы была необходимость их секционировать.

### Определение типа секционирования
Будем секционировать таблицу bookings по дате с типом секционирования range. Посмотрим распределение записей по годам и месяцам:
```
demo=# select date_part('year',book_date),date_part('month',book_date),count(*)
demo-# from bookings.bookings
demo-# group by date_part('year',book_date),date_part('month',book_date)
demo-# order by date_part('year',book_date),date_part('month',book_date);
 date_part | date_part | count
-----------+-----------+--------
      2015 |         9 |  15997
      2015 |        10 | 169511
      2015 |        11 | 165472
      2015 |        12 | 170937
      2016 |         1 | 170965
      2016 |         2 | 160259
      2016 |         3 | 171229
      2016 |         4 | 165514
      2016 |         5 | 171346
      2016 |         6 | 165506
      2016 |         7 | 170956
      2016 |         8 | 170749
      2016 |         9 | 166285
      2016 |        10 |  76384
(14 rows)
```
Поскольку в среднем за меся около 160000 строк, то можно сделать секции по кварталам, 
при этом сентябрь 2015 года можно включить в первую секцию, поскольку там очень мало строк, и октябрь 2016 включим в последнюю.
### Создание секционированной таблицы
Создадим главную таблицу:
```
demo=# CREATE TABLE bookings.bookings_new
demo-# (
demo(#     book_ref character(6) COLLATE pg_catalog."default" NOT NULL,
demo(#     book_date timestamp with time zone NOT NULL,
demo(#     total_amount numeric(10,2) NOT NULL,
demo(#     CONSTRAINT bookings_pkey_new PRIMARY KEY (book_ref, book_date)
demo(# ) PARTITION BY RANGE (book_date);
CREATE TABLE
```
Создадим секции:  
```
demo=# CREATE TABLE bookings.bookings_new_2015_4 PARTITION OF bookings.bookings_new FOR VALUES FROM ('2015-09-01') TO ('2016-01-01');
');CREATE TABLE
demo=# CREATE TABLE bookings.bookings_new2016_1 PARTITION OF bookings.bookings_new FOR VALUES FROM ('2016-01-01') TO ('2016-04-01');
CREATE TABLE
demo=# CREATE TABLE bookings.bookings_new_2016_2 PARTITION OF bookings.bookings_new FOR VALUES FROM ('2016-04-01') TO ('2016-07-01');
CREATE TABLE
demo=# CREATE TABLE bookings.bookings_new_2016_3 PARTITION OF bookings.bookings_new FOR VALUES FROM ('2016-07-01') TO ('2016-11-01');
CREATE TABLE
demo=#
```
### Миграция данных
Проводим миграцию данных:
```
demo=# INSERT INTO bookings.bookings_new
demo-# OVERRIDING SYSTEM VALUE
demo-# SELECT * FROM bookings.bookings;
INSERT 0 2111110
```
Проверяем корректность:  
Первая секция:
```
demo=# select date_part('year',book_date),date_part('month',book_date),count(*)
demo-# from bookings.bookings_new_2015_4
demo-# group by date_part('year',book_date),date_part('month',book_date)
demo-# order by date_part('year',book_date),date_part('month',book_date);
 date_part | date_part | count
-----------+-----------+--------
      2015 |         9 |  15997
      2015 |        10 | 169511
      2015 |        11 | 165472
      2015 |        12 | 170937
(4 rows)
```
Вторая секция:
```
demo=# select date_part('year',book_date),date_part('month',book_date),count(*)
demo-# from bookings.bookings_new2016_1
demo-# group by date_part('year',book_date),date_part('month',book_date)
demo-# order by date_part('year',book_date),date_part('month',book_date);
 date_part | date_part | count
-----------+-----------+--------
      2016 |         1 | 170965
      2016 |         2 | 160259
      2016 |         3 | 171229
(3 rows)
```
Третья секция:
```
demo=# select date_part('year',book_date),date_part('month',book_date),count(*)
demo-# from bookings.bookings_new_2016_2
demo-# group by date_part('year',book_date),date_part('month',book_date)
demo-# order by date_part('year',book_date),date_part('month',book_date);
 date_part | date_part | count
-----------+-----------+--------
      2016 |         4 | 165514
      2016 |         5 | 171346
      2016 |         6 | 165506
(3 rows)
```
Четвертая секция:
```
demo=# select date_part('year',book_date),date_part('month',book_date),count(*)
demo-# from bookings.bookings_new_2016_3
demo-# group by date_part('year',book_date),date_part('month',book_date)
demo-# order by date_part('year',book_date),date_part('month',book_date);
 date_part | date_part | count
-----------+-----------+--------
      2016 |         7 | 170956
      2016 |         8 | 170749
      2016 |         9 | 166285
      2016 |        10 |  76384
(4 rows)
```
Проверка на общее количество записей:
```
demo=# select count(*)
demo-# from bookings.bookings;
om bookings.bookings_new;  count
---------
 2111110
(1 row)

demo=# select count(*)
demo-# from bookings.bookings_new;
  count
---------
 2111110
(1 row)
```
Видим, что не потеряли и одной записи и все записи корректно распределились по секциям.
### Оптимизация запросов
При создании секционированной таблицы сразу было обращено внимание на то, что есть PRIMARY KEY по полю book_ref, его было решено сохранить. Других индексов на данной таблицы смысла делать нет.
Смотрим план запроса до секционирования:

```
EXPLAIN ANALYZE
select * 
from bookings.bookings
where book_date = '2015-10-06';
                                                        QUERY PLAN
--------------------------------------------------------------------------------------------------------------------------
 Gather  (cost=1000.00..25442.86 rows=5 width=21) (actual time=5.869..305.417 rows=3 loops=1)
   Workers Planned: 2
   Workers Launched: 2
   ->  Parallel Seq Scan on bookings  (cost=0.00..24442.36 rows=2 width=21) (actual time=147.397..243.408 rows=1 loops=3)
         Filter: (book_date = '2015-10-06 00:00:00+00'::timestamp with time zone)
         Rows Removed by Filter: 703702
 Planning Time: 0.049 ms
 Execution Time: 305.435 ms
```
План запроса после секционирования:
```
EXPLAIN ANALYZE
select *
from bookings.bookings_new
where book_date = '2015-10-06';
                                                                  QUERY PLAN
----------------------------------------------------------------------------------------------------------------------------------------------
 Gather  (cost=1000.00..7043.82 rows=5 width=21) (actual time=1.412..116.140 rows=3 loops=1)
   Workers Planned: 2
   Workers Launched: 2
   ->  Parallel Seq Scan on bookings_new_2015_4 bookings_new  (cost=0.00..6043.32 rows=2 width=21) (actual time=8.953..59.946 rows=1 loops=3)
         Filter: (book_date = '2015-10-06 00:00:00+00'::timestamp with time zone)
         Rows Removed by Filter: 173971
 Planning Time: 0.121 ms
 Execution Time: 116.158 ms
(8 rows)
```
Видим, что идет обращение только к одной секции и время выполнения запроса заметно уменьшилось.

### Тестирование решения:  
Проверяем вставку данных:
```
demo=# insert into bookings.bookings_new(book_ref,book_date,total_amount)
demo-# values('FF0000','2015-09-30'::date,10000);
INSERT 0 1
demo=# select *
demo-# from bookings.bookings_new_2015_4
demo-# where book_ref='FF0000';
 book_ref |       book_date        | total_amount
----------+------------------------+--------------
 FF0000   | 2015-09-30 00:00:00+00 |     10000.00
(1 row)
```
Видим, что запись вставилась корректно.
Изменение данных:
```
demo=# update bookings.bookings_new
demo-# set total_amount=20000
demo-# where book_ref='FF0000';
UPDATE 1
demo=# select *
demo-# from bookings.bookings_new
demo-# where book_ref='FF0000';
 book_ref |       book_date        | total_amount
----------+------------------------+--------------
 FF0000   | 2015-09-30 00:00:00+00 |     20000.00
(1 row)

demo=#
```
Изменение так же выполнилось корректно.  

Удаление данных:  
```
demo=# delete from bookings.bookings_new where book_ref='FF0000';
DELETE 1
demo=# select * from bookings.bookings_new where book_ref='FF0000';
 book_ref | book_date | total_amount
----------+-----------+--------------
(0 rows)
```
Видим, что запись удалена.

Посмотрим как ведут себя запросы с разными датами:
```
demo=# EXPLAIN ANALYZE
demo-# select *
demo-# from bookings.bookings_new
demo-# where book_date = '2016-02-06';
                                                                  QUERY PLAN
----------------------------------------------------------------------------------------------------------------------------------------------
 Gather  (cost=1000.00..6818.44 rows=5 width=21) (actual time=33.739..119.556 rows=7 loops=1)
   Workers Planned: 2
   Workers Launched: 2
   ->  Parallel Seq Scan on bookings_new2016_1 bookings_new  (cost=0.00..5817.94 rows=2 width=21) (actual time=11.918..55.414 rows=2 loops=3)
         Filter: (book_date = '2016-02-06 00:00:00+00'::timestamp with time zone)
         Rows Removed by Filter: 167482
 Planning Time: 0.094 ms
 Execution Time: 119.576 ms
(8 rows)

demo=# EXPLAIN ANALYZE
demo-# select *
demo-# from bookings.bookings_new
demo-# where book_date = '2016-09-06';
                                                                  QUERY PLAN
-----------------------------------------------------------------------------------------------------------------------------------------------
 Gather  (cost=1000.00..7767.11 rows=5 width=21) (actual time=57.637..133.249 rows=2 loops=1)
   Workers Planned: 2
   Workers Launched: 2
   ->  Parallel Seq Scan on bookings_new_2016_3 bookings_new  (cost=0.00..6766.61 rows=2 width=21) (actual time=37.861..58.170 rows=1 loops=3)
         Filter: (book_date = '2016-09-06 00:00:00+00'::timestamp with time zone)
         Rows Removed by Filter: 194791
 Planning Time: 0.070 ms
 Execution Time: 133.264 ms
(8 rows)

demo=# EXPLAIN ANALYZE
demo-# select *
demo-# from bookings.bookings_new
demo-# where book_date = '2016-04-06';
                                                                  QUERY PLAN
-----------------------------------------------------------------------------------------------------------------------------------------------
 Gather  (cost=1000.00..6816.99 rows=5 width=21) (actual time=83.623..126.556 rows=2 loops=1)
   Workers Planned: 2
   Workers Launched: 2
   ->  Parallel Seq Scan on bookings_new_2016_2 bookings_new  (cost=0.00..5816.49 rows=2 width=21) (actual time=51.492..64.584 rows=1 loops=3)
         Filter: (book_date = '2016-04-06 00:00:00+00'::timestamp with time zone)
         Rows Removed by Filter: 167455
 Planning Time: 0.086 ms
 Execution Time: 126.572 ms
(8 rows)
```
Видим, что все запросы идут в свои секции.
