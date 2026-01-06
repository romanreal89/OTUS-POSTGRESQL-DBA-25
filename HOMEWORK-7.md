## Работа с журналами
### Запуск PGBENCH:
```
root@otus-PC:~$ sudo -u postgres  pgbench -P 60 -T 600
pgbench (18.1 (Ubuntu 18.1-1.pgdg24.04+2))
starting vacuum...end.
progress: 60.0 s, 147.4 tps, lat 6.827 ms stddev 8.247, 0 failed
progress: 120.0 s, 164.3 tps, lat 5.012 ms stddev 7.853, 0 failed
progress: 180.0 s, 211.5 tps, lat 4.016 ms stddev 6.139, 0 failed
progress: 240.0 s, 168.1 tps, lat 5.317 ms stddev 6.354, 0 failed
progress: 300.0 s, 171.1 tps, lat 5.502 ms stddev 7.233, 0 failed
progress: 360.0 s, 169.9 tps, lat 5.451 ms stddev 7.464, 0 failed
progress: 420.0 s, 240.1 tps, lat 4.150 ms stddev 5.713, 0 failed
progress: 480.0 s, 214.2 tps, lat 4.831 ms stddev 6.448, 0 failed
progress: 540.0 s, 118.1 tps, lat 7.119 ms stddev 8.719, 0 failed
progress: 600.0 s, 135.4 tps, lat 6.341 ms stddev 9.315, 0 failed
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 1
number of threads: 1
maximum number of tries: 1
duration: 600 s
number of transactions actually processed: 105579
number of failed transactions: 0 (0.000%)
latency average = 5.673 ms
latency stddev = 7.574 ms
initial connection time = 6.275 ms
tps = 174.942019 (without initial connection time)
```
### Анализ журналов: 
В моем случае создавалось не более 6 файлов каждый размером около 16MB. 
Некоторые файлы были созданы более чем за 30 секунд, это можно объяснить тем, 
что у нас была выставлено очень частое создание контрольных точек и 
при нагрузке они не всегда успевают создаваться по времени настройки.

### Запуск PGBENCH d асинхронном режиме

```
root@otus-PC:~$ sudo -u postgres  pgbench -P 60 -T 600
pgbench (18.1 (Ubuntu 18.1-1.pgdg24.04+2))
starting vacuum...end.
progress: 60.0 s, 589.2 tps, lat 1.726 ms stddev 1.743, 0 failed
progress: 120.0 s, 553.7 tps, lat 1.815 ms stddev 1.773, 0 failed
progress: 180.0 s, 928.3 tps, lat 0.920 ms stddev 1.638, 0 failed
progress: 240.0 s, 618.5 tps, lat 2.389 ms stddev 2.247, 0 failed
progress: 300.0 s, 658.1 tps, lat 1.992 ms stddev 1.734, 0 failed
progress: 360.0 s, 1040.6 tps, lat 0.976 ms stddev 1.456, 0 failed
progress: 420.0 s, 583.5 tps, lat 2.067 ms stddev 1.813, 0 failed
progress: 480.0 s, 576.7 tps, lat 1.663 ms stddev 1.763, 0 failed
progress: 540.0 s, 880.6 tps, lat 1.769 ms stddev 1.522, 0 failed
progress: 600.0 s, 591.8 tps, lat 1.433 ms stddev 1.623, 0 failed
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 1
number of threads: 1
maximum number of tries: 1
duration: 600 s
number of transactions actually processed: 406683
number of failed transactions: 0 (0.000%)
latency average = 1.485 ms
latency stddev = 1.771 ms
initial connection time = 6.514 ms
tps = 657.803661 (without initial connection time)
```
В асинхронном режиме tps в несколько раз больше поскольку запись в журнал происходит гораздо реже.

### Включаем контрольную сумму

```
postgres=# show data_checksums;
 data_checksums
----------------
 on
(1 row)
```
### Создаем таблицу и вставляем данные:
```
postgres=# CREATE TABLE test_check_sum(id serial,tname char(100));
CREATE TABLE
postgres=# INSERT INTO test_check_sum(tname) SELECT 'testdata' FROM generate_series(1,100000);
INSERT 0 100000
postgres=#  SELECT pg_relation_filepath('test_check_sum');
```
### Находим таблицу:
```
postgres=#  SELECT pg_relation_filepath('test_check_sum');
 pg_relation_filepath
----------------------
 base/5/16828
(1 row)
```

### Меняем содержимое:

```
Стоп кластера:
root@otus-PC:/var/lib/postgresql/18$ sudo pg_ctlcluster 18 main stop
```
Добавлем символы.
```
Старт кластера:
root@otus-PC:/var/lib/postgresql/18$ sudo pg_ctlcluster 18 main start
```
### Выполняем запрос:

```
postgres=# select * from test_check_sum;
WARNING:  page verification failed, calculated checksum 15121 but expected 12624
ERROR:  invalid page in block 0 of relation base/5/16828
```
Произошла ошибка при проверке контрольной суммы. Для того, чтобы пропускать такие ошибки можно выполнить команду:  
alter system set ignore_checksum_failure = on;
