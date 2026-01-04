### Бэкапы

#### Создание БД и таблицы
```
root@otus-PC:/home/otus# sudo -u postgres psql
psql (18.1 (Ubuntu 18.1-1.pgdg24.04+2))
Type "help" for help.

postgres=# CREATE DATABASE db4backup;
CREATE DATABASE
postgres=# \c db4backup
You are now connected to database "db4backup" as user "postgres".
db4backup=# CREATE SCHEMA demo;
CREATE SCHEMA
db4backup=# CREATE TABLE demo.users (id SERIAL PRIMARY KEY,name VARCHAR(50),email VARCHAR(100));
CREATE TABLE
db4backup=# INSERT INTO demo.users (name, email)
```
#### Заполним данными
```
SELECT 'User' || g.id, 'user' || g.id || '@example.com'
FROM generate_series(1, 100) AS g(id);
INSERT 0 100
```
Проверим
```
db4backup=# SELECT * FROM demo.users LIMIT 15;
 id |  name  |       email
----+--------+--------------------
  1 | User1  | user1@example.com
  2 | User2  | user2@example.com
  3 | User3  | user3@example.com
  4 | User4  | user4@example.com
  5 | User5  | user5@example.com
  6 | User6  | user6@example.com
  7 | User7  | user7@example.com
  8 | User8  | user8@example.com
  9 | User9  | user9@example.com
 10 | User10 | user10@example.com
 11 | User11 | user11@example.com
 12 | User12 | user12@example.com
 13 | User13 | user13@example.com
 14 | User14 | user14@example.com
 15 | User15 | user15@example.com
(15 rows)

db4backup=# SELECT count(*) FROM demo.users;
 count
-------
   100
(1 row)

```

### Логический бэкап

Создадим директорию и настроем доступы

```
root@otus-PC:/home/otus# mkdir /backups
root@otus-PC:/home/otus# chmod 700 /backups
root@otus-PC:/home/otus# chown postgres:postgres /backups
root@otus-PC:/home/otus# sudo -u postgres psql
psql (18.1 (Ubuntu 18.1-1.pgdg24.04+2))
Type "help" for help.

postgres=# \c db4backup
You are now connected to database "db4backup" as user "postgres".
db4backup=# COPY demo.users TO '/backups/users_backup.csv' WITH CSV HEADER;
COPY 100
db4backup=# \q
```
Проверим наличие файла
```
root@otus-PC:/home/otus# cd /backups
root@otus-PC:/backups# ls
users_backup.csv
```
Создадим вторую таблицу
```
root@otus-PC:/backups# sudo -u postgres psql
psql (18.1 (Ubuntu 18.1-1.pgdg24.04+2))
Type "help" for help.

postgres=# \c db4backup
You are now connected to database "db4backup" as user "postgres".
db4backup=# CREATE TABLE demo.users_restored ( id SERIAL PRIMARY KEY, name VARCHAR(50), email VARCHAR(100));
CREATE TABLE
```
Проверим, что она пуста
```
db4backup=# select * from demo.users_restored;
 id | name | email
----+------+-------
(0 rows)
```
Восстанавливаем в нее данные из файла и проверяем
```
db4backup=# COPY demo.users_restored FROM '/backups/users_backup.csv' WITH CSV HEADER;
COPY 100
db4backup=# select count(*) from demo.users_restored;
 count
-------
   100
(1 row)
```

### Физический бэкап

Создадим бэкап, используя pg_dump. Перед этим и после проверим, что файла дампа не было и он появился.
```
root@otus-PC:/backups# ls
users_backup.csv
root@otus-PC:/backups# pg_dump -U postgres -d db4backup \
  -t 'demo.users' \
  -t 'demo.users_restored' \
  -Fc \
  -f /backups/backup_custom.dump
root@otus-PC:/backups# sudo -i -u postgres pg_dump -U postgres -d db4backup   -t 'demo.users'   -t 'demo.users_restored'   -Fc   -f /backups/backup_custom.dump
root@otus-PC:/backups# ls
backup_custom.dump  users_backup.csv
```
Создадим новую бд
```
root@otus-PC:/backups# sudo -u postgres psql
psql (18.1 (Ubuntu 18.1-1.pgdg24.04+2))
Type "help" for help.

postgres=# CREATE DATABASE db4restore;
CREATE DATABASE
postgres=# \q
```
Восстановим в нее данные из бэкапа и проверим
```
root@otus-PC:/backups#  sudo -i -u postgres pg_restore -U postgres \
  -d db4restore \
  -t 'demo.users_restored' \
  /backups/backup_custom.dump
root@otus-PC:/backups# sudo -u postgres psql
psql (18.1 (Ubuntu 18.1-1.pgdg24.04+2))
Type "help" for help.

postgres=# \c db4restore
You are now connected to database "db4restore" as user "postgres".
db4restore=# select count(*) from demo.users_restored;
 count
-------
   100
(1 row)
```