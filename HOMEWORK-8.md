## Механизм блокировок
### Настраиваем сервер:  
```
Настройки pg_config
log_lock_waits = on
deadlock_timeout = 200ms
```
Перечитываем настройки и перегружаем кластер:  
```
sudo systemctl daemon-reload
sudo pg_ctlcluster 18 main restart

root@otus-PC:~$ pg_lsclusters
Ver Cluster Port Status Owner    Data directory              Log file
18  main    5432 online postgres /var/lib/postgresql/18/main /var/log/postgresql/postgresql-18-main.log
```

Создаем БД для работы, в ней таблицу и заполняем данными:  
```
postgres=# create database locks;
CREATE DATABASE
postgres=# \c locks
You are now connected to database "locks" as user "postgres".
locks=# CREATE TABLE accounts(
locks(#   acc_no integer PRIMARY KEY,
locks(#   amount numeric
locks(# );
CREATE TABLE
locks=# INSERT INTO accounts VALUES (1,1000.00), (2,2000.00), (3,3000.00);
INSERT 0 3
```

### Делаем симуляцию блокировки:

В певрой сессии делаем апдейт без коммита:
```
locks=# \set AUTOCOMMIT off
locks=# begin;
BEGIN
locks=*# update accounts set amount=4000.00 where acc_no=2;
UPDATE 1
```
Во второй сессии делаем такой же апдейт:  
```
locks=# update accounts set amount=4000.00 where acc_no=2;
```
Сессия повисает, через некоторое время отменяем операцию:  
```
Cancel request sent
ERROR:  canceling statement due to user request
CONTEXT:  while updating tuple (0,2) in relation "accounts"
```

Провераем логи:
```
root@otus-PC:~$ sudo cat /var/log/postgresql/postgresql-18-main.log
```
Видим строки, которые говорят о наличии блокировок:
```
2026-01-07 12:27:50.050 UTC [27918] postgres@locks LOG:  process 27918 still waiting for ShareLock on transaction 738 after 200.319 ms
2026-01-07 12:27:50.050 UTC [27918] postgres@locks DETAIL:  Process holding the lock: 27749. Wait queue: 27918.
2026-01-07 12:27:50.050 UTC [27918] postgres@locks CONTEXT:  while updating tuple (0,2) in relation "accounts"
2026-01-07 12:27:50.050 UTC [27918] postgres@locks STATEMENT:  update accounts set amount=4000.00 where acc_no=2;
2026-01-07 12:28:12.085 UTC [27918] postgres@locks ERROR:  canceling statement due to user request
2026-01-07 12:28:12.085 UTC [27918] postgres@locks CONTEXT:  while updating tuple (0,2) in relation "accounts"
2026-01-07 12:28:12.085 UTC [27918] postgres@locks STATEMENT:  update accounts set amount=4000.00 where acc_no=2;
```
### Update в 3-х сессиях:

Запускаем update в первой сессии:
```
locks=# \set AUTOCOMMIT off
locks=# update accounts set amount=4000.00 where acc_no=1;
UPDATE 1
```
Затем такую команду еще в двух сессиях:
```
locks=# update accounts set amount=4000.00 where acc_no=1;
```
Смотрим pg_locks:

![NEWVOLUME](https://github.com/romanreal89/OTUS-POSTGRESQL-DBA-25/blob/main/blocks.JPG?raw=true)

Где:  
16389 | accounts  
16394 | accounts_pkey  
12073 | pg_locks   

Посмотрим с учетом зависимостей:  
![NEWVOLUME](https://github.com/romanreal89/OTUS-POSTGRESQL-DBA-25/blob/main/blocks2.JPG?raw=true)

Видим, что процесс 27749 заблокировал таблицу в RowExclusiveLock, процесс 28225 ожидает процесс 27749 с RowExclusiveLock, а процесс 27918 ожидает 28225 с RowExclusiveLock.

### Анализ блокровок по логам.

Воспроизводим блокировки:  
Запускаем update в первой сессии:
```
locks=# \set AUTOCOMMIT off
locks=# update accounts set amount=4000.00 where acc_no=3;
UPDATE 1
```
Затем такую команду еще в двух сессиях:
```
locks=# update accounts set amount=4000.00 where acc_no=3;
```
Смортим логи:  
```
2025-03-13 16:24:51.336 UTC [28225] postgres@locks LOG:  process 28225 acquired ShareLock on transaction 740 after 1811671.281 ms
2025-03-13 16:24:51.336 UTC [28225] postgres@locks CONTEXT:  while updating tuple (0,1) in relation "accounts"
2025-03-13 16:24:51.336 UTC [28225] postgres@locks STATEMENT:  update accounts set amount=4000.00 where acc_no=1;
2025-03-13 16:24:51.341 UTC [27918] postgres@locks LOG:  process 27918 acquired ExclusiveLock on tuple (0,1) of relation 16389 of database 16388 after 1807303.951 ms
2025-03-13 16:24:51.341 UTC [27918] postgres@locks STATEMENT:  update accounts set amount=4000.00 where acc_no=1;
2025-03-13 16:27:10.452 UTC [27918] postgres@locks LOG:  process 27918 still waiting for ShareLock on transaction 743 after 200.335 ms
2025-03-13 16:27:10.452 UTC [27918] postgres@locks DETAIL:  Process holding the lock: 27749. Wait queue: 27918.
2025-03-13 16:27:10.452 UTC [27918] postgres@locks CONTEXT:  while updating tuple (0,3) in relation "accounts"
2025-03-13 16:27:10.452 UTC [27918] postgres@locks STATEMENT:  update accounts set amount=4000.00 where acc_no=3;
2025-03-13 16:27:16.769 UTC [28225] postgres@locks LOG:  process 28225 still waiting for ExclusiveLock on tuple (0,3) of relation 16389 of database 16388 after 200.415 ms
2025-03-13 16:27:16.769 UTC [28225] postgres@locks DETAIL:  Process holding the lock: 27918. Wait queue: 28225.
2025-03-13 16:27:16.769 UTC [28225] postgres@locks STATEMENT:  update accounts set amount=4000.00 where acc_no=3;
2025-03-13 16:27:21.821 UTC [26824] LOG:  checkpoint starting: time
```
Из логов видно, что первым заблокировал процесс 27918, затем уже процесс 28225 и он ожидает 27918, и на конец процесс 27749 ожидает 27918.

### Могут ли две транзакции, выполняющие единственную команду UPDATE одной и той же таблицы (без where), заблокировать друг друга?

Ответ нет, поскольку сначала таблицу полностью заблокирует первая транзакция,а вторая будет либо ожидать, либо отвалитс по deadlock detected.
