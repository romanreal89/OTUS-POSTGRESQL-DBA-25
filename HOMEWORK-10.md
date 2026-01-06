## Работа с индексами
### Создание простого индекса
#### Подготовка
Создаем БД: 
```
postgres=# CREATE DATABASE db4indexes;
CREATE DATABASE
```
Создаем и заполняем таблицу:  
```
postgres=# \c db4indexes;
You are now connected to database "db4indexes" as user "postgres".
db4indexes=# CREATE TABLE products(
db4indexes(#     product_id   integer,
db4indexes(#     brand        char(1),
db4indexes(#     gender       char(1),
db4indexes(#     price        integer,
db4indexes(#     is_available boolean
db4indexes(# );
CREATE TABLE
db4indexes=# WITH random_data AS (
db4indexes(#     SELECT
db4indexes(#     num,
db4indexes(#     random() AS rand1,
db4indexes(#     random() AS rand2,
db4indexes(#     random() AS rand3
db4indexes(#     FROM generate_series(1, 10000000) AS s(num)
db4indexes(# )
db4indexes-# INSERT INTO products
db4indexes-#     (product_id, brand, gender, price, is_available)
db4indexes-# SELECT
db4indexes-#     random_data.num,
db4indexes-#     chr((32 + random_data.rand1 * 94)::integer),
db4indexes-#     case when random_data.num % 2 = 0 then 'М' else 'Ж' end,
db4indexes-#     (random_data.rand2 * 100)::integer,
db4indexes-#     random_data.rand3 < 0.01
db4indexes-#     FROM random_data
db4indexes-#     ORDER BY random();
INSERT 0 10000000
```
#### Работа с простым индексом:  
Смотрим на план до создания индекса:
```
db4indexes=# EXPLAIN
db4indexes-# SELECT * FROM products
db4indexes-#     WHERE product_id = 1878;
                                   QUERY PLAN
---------------------------------------------------------------------------------
 Gather  (cost=1000.00..116779.03 rows=1 width=14)
   Workers Planned: 2
   ->  Parallel Seq Scan on products  (cost=0.00..115778.93 rows=1 width=14)
         Filter: (product_id = 1878)
 JIT:
   Functions: 2
   Options: Inlining false, Optimization false, Expressions true, Deforming true
(7 rows)
```
```
db4indexes=# EXPLAIN
db4indexes-# SELECT * FROM products
db4indexes-#     WHERE product_id < 10099;
                                   QUERY PLAN
---------------------------------------------------------------------------------
 Gather  (cost=1000.00..117874.03 rows=10951 width=14)
   Workers Planned: 2
   ->  Parallel Seq Scan on products  (cost=0.00..115778.93 rows=4563 width=14)
         Filter: (product_id < 10099)
 JIT:
   Functions: 2
   Options: Inlining false, Optimization false, Expressions true, Deforming true
(7 rows)
```
Видим, что в обоих случаях идет последовательное сканирование таблицы: Seq Scan on products  
Создаем индекс: 
```
db4indexes=# CREATE INDEX idx_products_product_id ON products(product_id);
```
Смотрим план: 
```
db4indexes=# EXPLAIN
db4indexes-# SELECT * FROM products
db4indexes-#     WHERE product_id = 1878;
                                       QUERY PLAN
-----------------------------------------------------------------------------------------
 Index Scan using idx_products_product_id on products  (cost=0.43..8.45 rows=1 width=14)
   Index Cond: (product_id = 1878)
(2 rows)
```
```
db4indexes=# EXPLAIN
SELECT * FROM products
    WHERE product_id < 10099;
                                         QUERY PLAN
--------------------------------------------------------------------------------------------
 Bitmap Heap Scan on products  (cost=223.20..30068.98 rows=11711 width=14)
   Recheck Cond: (product_id < 10099)
   ->  Bitmap Index Scan on idx_products_product_id  (cost=0.00..220.27 rows=11711 width=0)
         Index Cond: (product_id < 10099)
(4 rows)
```
Видим теперь, что для первого случая идет Index Scanа, 
во втором случае сначала идет Bitmap Index Scan, 
для поиска адрес строк, а уже потом идет Bitmap Heap Scan для получения нужных строк с результатом поиска.
B-tree индексы дают ускорение при поиске данных, которые можно отсортировать.
#### Работа с полнотекстовым поиском: 
Создаем таблицу:  
```
CREATE TABLE documents (
    title    varchar(64),
    metadata jsonb,
    contents text
);
```
Вставляем данные:  
```
db4indexes=# INSERT INTO documents
db4indexes-#     (title, metadata, contents)
db4indexes-# VALUES
db4indexes-#     ( 'Document 1',
db4indexes(#       '{"author": "James",  "tags": ["legal", "real estate"]}',
db4indexes(#       'This is a legal document about real estate.' ),
db4indexes-#     ( 'Document 2',
db4indexes(#       '{"author": "Ivan",  "tags": ["finance", "legal"]}',
db4indexes(#       'Financial statements should be verified.' ),
db4indexes-#     ( 'Document 3',
db4indexes(#       '{"author": "Sofi",  "tags": ["health", "nutrition"]}',
db4indexes(#       'Regular exercise promotes better health.' ),
db4indexes-#     ( 'Document 4',
db4indexes(#       '{"author": "Alice", "tags": ["travel", "adventure"]}',
db4indexes(#       'Mountaineering requires careful preparation.' ),
db4indexes-#     ( 'Document 5',
db4indexes(#       '{"author": "Tom",   "tags": ["legal", "contracts"]}',
db4indexes(#       'Contracts are binding legal documents.' ),
db4indexes-#     ( 'Document 6',
db4indexes(#        '{"author": "Alex",  "tags": ["legal", "family law"]}',
db4indexes(#        'Family law addresses diverse issues.' ),
db4indexes-#     ( 'Document 7',
db4indexes(#       '{"author": "James",  "tags": ["technology", "innovation"]}',
db4indexes(#       'Tech innovations are changing the world.' ),
db4indexes-# ( 'Document 8',
db4indexes(#       '{"author": "James",  "tags": ["technology", "contracts"]}',
db4indexes(#       'Tech innovations are changing the world.' ),
db4indexes-# ( 'Document 9',
db4indexes(#       '{"author": "Tom",  "tags": ["legal", "innovation"]}',
db4indexes(#       'Tech innovations are changing the world.' ),
db4indexes-# ( 'Document 10',
db4indexes(#       '{"author": "James",  "tags": ["finance", "family law"]}',
db4indexes(#       'Tech innovations are changing the world.' ),
db4indexes-# ( 'Document 11',
db4indexes(#       '{"author": "James",  "tags": ["technology", "innovation"]}',
db4indexes(#       'Tech innovations are changing the world.' ),
db4indexes-# ( 'Document 12',
db4indexes(#       '{"author": "Sofi",  "tags": ["legal", "family law"]}',
db4indexes(#       'Tech innovations are changing the world.' ),
db4indexes-# ( 'Document 13',
db4indexes(#       '{"author": "Alex",  "tags": ["technology", "contracts"]}',
db4indexes(#       'Tech innovations are changing the world.' ),
db4indexes-# ( 'Document 14',
db4indexes(#       '{"author": "Tom",  "tags": ["legal", "innovation"]}',
db4indexes(#       'Tech innovations are changing the world.' ),
db4indexes-# ( 'Document 15',
db4indexes(#       '{"author": "James",  "tags": ["finance", "innovation"]}',
db4indexes(#       'Tech innovations are changing the world.' ),
db4indexes-# ( 'Document 16',
db4indexes(#       '{"author": "Alex",  "tags": ["legal", "contracts"]}',
db4indexes(#       'Tech innovations are changing the world.' ),
db4indexes-# ( 'Document 17',
db4indexes(#       '{"author": "Tom",  "tags": ["technology", "family law"]}',
db4indexes(#       'Tech innovations are changing the world.' ),
db4indexes-# ( 'Document 18',
db4indexes(#       '{"author": "Tom",  "tags": ["technology", "contracts"]}',
db4indexes(#       'Tech innovations are changing the world.' ),
db4indexes-# ( 'Document 19',
db4indexes(#       '{"author": "James",  "tags": ["legal", "family law"]}',
db4indexes(#       'Tech innovations are changing the world.' ),
db4indexes-# ( 'Document 20',
db4indexes(#       '{"author": "Alex",  "tags": ["finance", "contracts"]}',
db4indexes(#       'Tech innovations are changing the world.' ),
db4indexes-# ( 'Document 21',
db4indexes(#       '{"author": "Tom",  "tags": ["legal", "innovation"]}',
db4indexes(#       'Tech innovations are changing the world.' );
INSERT 0 21
```
Смотрим план с полнотекстовым поиском:
```
db4indexes=# EXPLAIN
db4indexes-# SELECT * FROM documents
db4indexes-#     WHERE contents like '%document%';
                         QUERY PLAN
------------------------------------------------------------
 Seq Scan on documents  (cost=0.00..14.25 rows=1 width=210)
   Filter: (contents ~~ '%document%'::text)
(2 rows)

```
Создаем GIN-индекс:  
```
db4indexes=# CREATE INDEX idx_documents_contents ON documents USING GIN(to_tsvector('english', contents));
CREATE INDEX
```
Смотрим план:
```
db4indexes=# EXPLAIN
db4indexes-# SELECT * FROM documents
db4indexes-#     WHERE to_tsvector('english', contents) @@ 'document';
                                          QUERY PLAN
----------------------------------------------------------------------------------------------
 Bitmap Heap Scan on documents  (cost=8.00..12.26 rows=1 width=210)
   Recheck Cond: (to_tsvector('english'::regconfig, contents) @@ '''document'''::tsquery)
   ->  Bitmap Index Scan on idx_documents_contents  (cost=0.00..8.00 rows=1 width=0)
         Index Cond: (to_tsvector('english'::regconfig, contents) @@ '''document'''::tsquery)
(4 rows)
```
Видим, что до создания индекса при запросе происходило полное сканирование таблицы, 
после создания индекса сначала идет Bitmap Index Scan, для поиска адрес строк, 
а уже потом идет Bitmap Heap Scan для получения нужных строк с результатом поиска.  
Для текстового поиска, согласно документации, предпочтительными являются GIN-индексы. 
GIN-индексы представляют собой инвертированные индексы, в которых могут содержаться значения с несколькими ключами.  
Будучи инвертированными индексами, они содержат записи для всех отдельных слов (лексем) с компактным списком мест их вхождений. 

#### Работа с индексом на часть таблицы:
Создаем индекс:
```
db4indexes=# CREATE INDEX idx_products_is_available_true ON products(is_available) WHERE is_available = true;
CREATE INDEX
```
Смотрим план, где идет поиск по индексируемому значению:  
```
db4indexes=# EXPLAIN
db4indexes-# SELECT * FROM products
db4indexes-#     WHERE is_available = true;
                                               QUERY PLAN
--------------------------------------------------------------------------------------------------------
 Index Scan using idx_products_is_available_true on products  (cost=0.29..10341.56 rows=96000 width=14)
(1 row)
```
Видим, что идет  Index Scan.  
Посмотрим другой запрос:  
```
SELECT * FROM products
    WHERE is_available = false;
                                   QUERY PLAN
---------------------------------------------------------------------------------
 Seq Scan on products  (cost=0.00..163695.00 rows=9904000 width=14)
   Filter: (NOT is_available)
 JIT:
   Functions: 2
   Options: Inlining false, Optimization false, Expressions true, Deforming true
(5 rows)
```
Видим,что идет последовательное сканирование таблицы: Seq Scan.
Частичные индексы индексируют лишь подмножество строк таблицы. Это позволяет экономить размер индексов и быстрее выполнять сканирование.   
Эффективно применять когда выборка по индексироуемому значению происходит очень часто, например в таблице используется поле sysactive со значениями Y/D/U. И чаще всего выбираеются данные со значением поля: Y.

#### Работа с индексом на несколько полей:  
Смотрим на план запросов до создания индексов:
```
db4indexes=# EXPLAIN
db4indexes-# SELECT * FROM products
db4indexes-#     WHERE product_id <= 10000
db4indexes-#     AND brand = 'a';
                                         QUERY PLAN
--------------------------------------------------------------------------------------------
 Bitmap Heap Scan on products  (cost=215.53..29903.56 rows=121 width=14)
   Recheck Cond: (product_id <= 10000)
   Filter: (brand = 'a'::bpchar)
   ->  Bitmap Index Scan on idx_products_product_id  (cost=0.00..215.50 rows=11609 width=0)
         Index Cond: (product_id <= 10000)
(5 rows)

db4indexes=# EXPLAIN
db4indexes-# SELECT * FROM products
db4indexes-#     WHERE product_id = 1780
db4indexes-#     AND brand <= 'a';
                                       QUERY PLAN
-----------------------------------------------------------------------------------------
 Index Scan using idx_products_product_id on products  (cost=0.43..8.46 rows=1 width=14)
   Index Cond: (product_id = 1780)
   Filter: (brand <= 'a'::bpchar)
(3 rows)
```
Создаем индексы:  
```
db4indexes=# CREATE INDEX idx_products_product_id_brand ON products(product_id, brand);
CREATE INDEX
db4indexes=# CREATE INDEX idx_products_brand_product_id ON products(brand, product_id);
CREATE INDEX
```

Смотрим планы запросов:
```
db4indexes=# EXPLAIN
db4indexes-# SELECT * FROM products
db4indexes-#     WHERE product_id <= 10000
db4indexes-#     AND brand = 'a';
                                          QUERY PLAN
----------------------------------------------------------------------------------------------
 Bitmap Heap Scan on products  (cost=5.68..475.67 rows=121 width=14)
   Recheck Cond: ((brand = 'a'::bpchar) AND (product_id <= 10000))
   ->  Bitmap Index Scan on idx_products_brand_product_id  (cost=0.00..5.64 rows=121 width=0)
         Index Cond: ((brand = 'a'::bpchar) AND (product_id <= 10000))
(4 rows)

db4indexes=# EXPLAIN
db4indexes-# SELECT * FROM products
db4indexes-#     WHERE product_id = 1780
db4indexes-#     AND brand <= 'a';
                                          QUERY PLAN
-----------------------------------------------------------------------------------------------
 Index Scan using idx_products_product_id_brand on products  (cost=0.43..8.46 rows=1 width=14)
   Index Cond: ((product_id = 1780) AND (brand <= 'a'::bpchar))
(2 rows)
```
Для первого случая теперь вместо ранее созданного индекса idx_products_product_id 
используется составной индекс idx_products_brand_product_id, причем сначала  
идет Bitmap Index Scan, для поиска адрес строк, а уже потом идет Bitmap Heap Scan для получения нужных строк с результатом поиска.
Для второго случая идет сканирование по индексу idx_products_product_id_brand.

Составные индексы применяются в случаях, когда часто идут выборки по условию из нескольких полей.
