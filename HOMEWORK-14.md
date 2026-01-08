### Репликация
#### Настраиваем первую ВМ

1. Заходим в PG:
```
root@otus-PC:~$ sudo -u postgres psql
psql (18.1 (Ubuntu 18.1-1.pgdg24.04+2))
Type "help" for help.

postgres=# \c test
You are now connected to database "test" as user "postgres".
test=#
```

2. Создаем таблицы:
```
test=# CREATE TABLE customers (
aracter varying(15),
    region character varying(15)test(#     customer_id character varying(5) NOT NULL,
test(#     company_name character varying(40) NOT NULL,
test(#     contact_name character varying(30),
test(#     contact_title character varying(30),
test(#     address character varying(60),
test(#     city character varying(15),
test(#     region character varying(15),
test(#     postal_code character varying(10),
test(#     country character varying(15),
test(#     phone character varying(24),
test(#     fax character varying(24)
test(# );
CREATE TABLE
CREATE TABLE
test=# CREATE TABLE orders (
test(#     order_id smallint NOT NULL,
test(#     customer_id character varying(5),
test(#     employee_id smallint,
test(#     order_date date,
test(#     required_date date,
test(#     shipped_date date,
test(#     ship_via smallint,
test(#     freight real,
test(#     ship_name character varying(40),
test(#     ship_address character varying(60),
test(#     ship_city character varying(15),
test(#     ship_region character varying(15),
test(#     ship_postal_code character varying(10),
test(#     ship_country character varying(15)
test(# );
CREATE TABLE
```
3. Создаем пользователя для репликации:  
```
test=# CREATE ROLE repluser WITH
test-# LOGIN
test-# NOSUPERUSER
test-# NOCREATEDB
test-# NOCREATEROLE
test-# INHERIT
test-# REPLICATION
test-# NOBYPASSRLS
test-# CONNECTION LIMIT -1
test-# PASSWORD '12345';
CREATE ROLE
```

4. Вносим настройки в postgresql.conf:
```
wal_level = logical
wal_log_hints = on
```
pg_hba.conf:
```
host replication all 0.0.0.0/0
```

Создаем публикацию:
```
postgres=# CREATE PUBLICATION publication01
postgres-#    FOR TABLE public.customers
postgres-#     WITH (publish = 'insert, update, delete, truncate', publish_via_partition_root = false);
WARNING:  wal_level is insufficient to publish logical changes
HINT:  Set wal_level to "logical" before creating subscriptions.
CREATE PUBLICATION
postgres=#
```
Перезагружаем и проверяем что работает:
```
root@otus-PC:~$ sudo pg_ctlcluster 18 main restart
root@otus-PC:~$ sudo -u postgres pg_lsclusters
Ver Cluster Port Status Owner    Data directory              Log file
18  main    5432 online postgres /var/lib/postgresql/18/main /var/log/postgresql/postgresql-18-main.log
```
5. После настройки 2 ВМ создаем подписку на таблицу orders на втором сервере.
#### Настраиваем вторую ВМ

Повторяем п. 1-4.

Перезагружаем:
```
root@otus-PC2:~$ sudo pg_ctlcluster 18 main restart
Warning: The unit file, source configuration file or drop-ins of postgresql@15-main.service changed on disk. Run 'systemctl daemon-reload' to reload units.
root@otus-PC2:~$ sudo systemctl daemon-reload
root@otus-PC2:~$ sudo -u postgres pg_lsclusters
Ver Cluster Port Status Owner    Data directory              Log file
18  main    5432 online postgres /var/lib/postgresql/18/main /var/log/postgresql/postgresql-18-main.log
```
Создаем публикацию:
```
test=# CREATE PUBLICATION publication02
test-#     FOR TABLE public.orders
test-#     WITH (publish = 'insert, update, delete, truncate', publish_via_partition_root = false);
CREATE PUBLICATION
``` 
Создаем подписку на 1 сервер для таблицы customers.
#### Настраиваем третью ВМ

Повторяем п. 1-3.

Создаем подписки на 1 сервер для таблицы customers и на 2 сервер для таблицы orders.

#### Проверяем репликацию
Вставляем данные на первом сервере в таблицу customers:   
![INSCAST](https://github.com/romanreal89/OTUS-POSTGRESQL-DBA-25/blob/main/data_cast.JPG?raw=true)

Проверяем таблицу на втором сервере:
![DATACAST2](https://github.com/romanreal89/OTUS-POSTGRESQL-DBA-25/blob/main/cast02.JPG?raw=true)

Проверяем таблицу на третьем сервере:
![DATACAST3](https://github.com/romanreal89/OTUS-POSTGRESQL-DBA-25/blob/main/cast_03.JPG?raw=true)

Вставляем данные на втором сервере в таблицу orders:  
![INSORD](https://github.com/romanreal89/OTUS-POSTGRESQL-DBA-25/blob/main/data_ord.JPG?raw=true)

Проверяем таблицу на первом сервере:
![DATAORD1](https://github.com/romanreal89/OTUS-POSTGRESQL-DBA-25/blob/main/orders_01.JPG?raw=true)

Проверяем таблицу на третьем сервере:
![DATAORD3](https://github.com/romanreal89/OTUS-POSTGRESQL-DBA-25/blob/main/orders_03.JPG?raw=true)

Видим, что данные подтянулись во все таблицы. Все работает успешно.