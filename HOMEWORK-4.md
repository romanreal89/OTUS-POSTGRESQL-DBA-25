## Логический уровень PostgreSQL
### Устновка PostgreSQL14 и проверка, что кластер работает.
#### Кластеры
Проверим текущие кластеры
```
root@otus-PC:/home/otus# pg_lsclusters
Ver Cluster Port Status Owner    Data directory             Log file
18  main    5432 online postgres /mnt/data/18/main          /var/log/postgresql/postgresql-18-main.log
```
Создадим новый
```
root@otus-PC:/home/otus# pg_createcluster 18 new
Creating new PostgreSQL cluster 18/new ...
/usr/lib/postgresql/18/bin/initdb -D /var/lib/postgresql/18/new --auth-local peer --auth-host scram-sha-256 --no-instructions
The files belonging to this database system will be owned by user "postgres".
This user must also own the server process.

The database cluster will be initialized with locale "en_US.UTF-8".
The default database encoding has accordingly been set to "UTF8".
The default text search configuration will be set to "english".

Data page checksums are enabled.

fixing permissions on existing directory /var/lib/postgresql/18/new ... ok
creating subdirectories ... ok
selecting dynamic shared memory implementation ... posix
selecting default "max_connections" ... 100
selecting default "shared_buffers" ... 128MB
selecting default time zone ... Europe/Moscow
creating configuration files ... ok
running bootstrap script ... ok
performing post-bootstrap initialization ... ok
syncing data to disk ... ok
```
Проверим текущие кластеры еще раз
```
root@otus-PC:/home/otus# pg_lsclusters
Ver Cluster Port Status Owner    Data directory             Log file
18  main    5432 online postgres /mnt/data/18/main          /var/log/postgresql/postgresql-18-main.log
18  new     5433 online postgres /var/lib/postgresql/18/new /var/log/postgresql/postgresql-18-new.log
```

Подключимся к новому кластеру
```
root@otus-PC:/home/otus# sudo -u postgres psql -p 5433
psql (18.1 (Ubuntu 18.1-1.pgdg24.04+2))
Type "help" for help.

postgres=#
```
#### Работа с доступами
Создаем бд и переподключаемся
```
root@otus-PC:/home/otus# sudo -u postgres psql -p 5433
psql (18.1 (Ubuntu 18.1-1.pgdg24.04+2))
Type "help" for help.

postgres=# create database testdb;
CREATE DATABASE
postgres=# \q

root@otus-PC:/home/otus# sudo -u postgres psql -p 5433 -d testdb
psql (18.1 (Ubuntu 18.1-1.pgdg24.04+2))
Type "help" for help.
```
Далее создаем схему, таблицу, делаем вставку
```
testdb=# create schema testnm;
CREATE SCHEMA
testdb=# CREATE TABLE testnm.t1 (c1 INTEGER);
CREATE TABLE
testdb=# INSERT INTO testnm.t1 (c1) VALUES (1);
INSERT 0 1
```
Теперь добавим роли, пользователя и доступы
```
testdb=# CREATE ROLE readonly;
CREATE ROLE
testdb=# GRANT CONNECT ON DATABASE testdb TO readonly;
GRANT
testdb=#
GRANT USAGE ON SCHEMA testnm TO readonly;
GRANT
testdb=# GRANT SELECT ON ALL TABLES IN SCHEMA testnm TO readonly;
GRANT
testdb=# CREATE USER testread WITH PASSWORD 'test123';
CREATE ROLE
testdb=# GRANT readonly TO testread;
GRANT ROLE
testdb=# \q
```
Пробуем подключиться из-под нового юзера
```
root@otus-PC:/home/otus# sudo -u postgres psql -p 5433 -d testdb -U testread -W
Password:
psql: error: connection to server on socket "/var/run/postgresql/.s.PGSQL.5433" failed: FATAL:  Peer authentication failed for user "testread"
```
Вылезла ошибка, зайдем и поменяем значение peer на md5  в конфиге авторизации
Пробуем еще раз.
```
root@otus-PC:/home/otus# sudo -u postgres psql -p 5433 -d testdb -U testread -W
Password:
psql (18.1 (Ubuntu 18.1-1.pgdg24.04+2))
Type "help" for help.
```
Пробуем прочитать данные
```
testdb=> SELECT * FROM testnm.t1;
 c1
----
  1
(1 row)
```
Попытка создать таблицу
```
testdb=> create table t2(c1 integer); insert into t2 values (2);
ERROR:  permission denied for schema public
LINE 1: create table t2(c1 integer);
                     ^
ERROR:  relation "t2" does not exist
LINE 1: insert into t2 values (2);
                    ^
```
Исправляем
```
testdb=> \q
root@otus-PC:/home/otus# sudo -u postgres psql -p 5433 -d testdb
psql (18.1 (Ubuntu 18.1-1.pgdg24.04+2))
Type "help" for help.

testdb=# GRANT CREATE ON  SCHEMA testnm TO readonly;
GRANT
testdb=# \q
root@otus-PC:/home/otus# sudo -u postgres psql -p 5433 -d testdb -U testread -W
Password:
psql (18.1 (Ubuntu 18.1-1.pgdg24.04+2))
Type "help" for help.
```
Пробуем еще раз
```
testdb=> create table testnm.t2(c1 integer); insert into testnm.t2 values (2);
CREATE TABLE
INSERT 0 1
```
#### Выводы
1. Сразу помню про полное имя при создании таблицы с учетом схемы.
2. Если создавать без учета схемы - будут ошибки, также стоит учитывать схему при написании запросов.