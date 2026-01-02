## Нагрузочное тестирование и тюнинг PostgreSQL  
### Создание и настройка VM

Будем использовать ВМ из предыдущих ДЗ.
ВМ создана на базе Proxmox Virtual Environment 8.4.14.

Первый запуск pgbench на дефолтных настройках
```
root@otus-PC:/home/otus# sudo -u postgres pgbench -c 50 -j 2 -P 10 -T 60 postgres
pgbench (18.1 (Ubuntu 18.1-1.pgdg24.04+2))
starting vacuum...end.
progress: 10.0 s, 230.2 tps, lat 209.439 ms stddev 299.825, 0 failed
progress: 20.0 s, 251.9 tps, lat 199.362 ms stddev 258.118, 0 failed
progress: 30.0 s, 252.3 tps, lat 199.868 ms stddev 296.063, 0 failed
progress: 40.0 s, 253.3 tps, lat 196.738 ms stddev 262.757, 0 failed
progress: 50.0 s, 256.7 tps, lat 195.831 ms stddev 201.142, 0 failed
progress: 60.0 s, 258.8 tps, lat 186.433 ms stddev 236.483, 0 failed
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 50
number of threads: 2
maximum number of tries: 1
duration: 60 s
number of transactions actually processed: 15082
number of failed transactions: 0 (0.000%)
latency average = 199.110 ms
latency stddev = 263.858 ms
initial connection time = 47.193 ms
tps = 250.709494 (without initial connection time)
```
Далее меняем следующие параметры в файле **postgresql.conf**
```
shared_buffers = 1GB
effective_io_concurrency = 200
fsync = off
synchronous_commit = off
full_page_writes = off
wal_buffers = 16MB
```
Рестартим интсанс. Требует перечитать конфиги. Выполняем.
```
root@otus-PC:/etc/postgresql/18/main# systemctl restart postgresql
Warning: The unit file, source configuration file or drop-ins of postgresql.service changed on disk. Run 'systemctl daemon-reload' to reload units.
root@otus-PC:/etc/postgresql/18/main# systemctl daemon-reload
root@otus-PC:/etc/postgresql/18/main# systemctl restart postgresql
```
Запускаем тот же тест еще раз.
```
root@otus-PC:/etc/postgresql/18/main# sudo -u postgres pgbench -c 50 -j 2 -P 10 -T 60 postgres
pgbench (18.1 (Ubuntu 18.1-1.pgdg24.04+2))
starting vacuum...end.
progress: 10.0 s, 161.9 tps, lat 301.244 ms stddev 317.840, 0 failed
progress: 20.0 s, 256.6 tps, lat 195.321 ms stddev 236.641, 0 failed
progress: 30.0 s, 313.3 tps, lat 158.924 ms stddev 194.811, 0 failed
progress: 40.0 s, 313.2 tps, lat 159.585 ms stddev 237.300, 0 failed
progress: 50.0 s, 315.8 tps, lat 159.068 ms stddev 187.912, 0 failed
progress: 60.0 s, 298.7 tps, lat 166.718 ms stddev 179.237, 0 failed
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 50
number of threads: 2
maximum number of tries: 1
duration: 60 s
number of transactions actually processed: 16645
number of failed transactions: 0 (0.000%)
latency average = 180.343 ms
latency stddev = 224.915 ms
initial connection time = 49.015 ms
tps = 276.865184 (without initial connection time)
```
Теперь поменяем настройки для чекпоинтов
```
checkpoint_timeout = 30min             
checkpoint_completion_target = 0.9
```
Перезапускаем.
Выполняем тест повторно.
```
root@otus-PC:/etc/postgresql/18/main# sudo -u postgres pgbench -c 50 -j 2 -P 10 -T 60 postgres
pgbench (18.1 (Ubuntu 18.1-1.pgdg24.04+2))
starting vacuum...end.
progress: 10.0 s, 295.8 tps, lat 165.050 ms stddev 171.204, 0 failed
progress: 20.0 s, 313.4 tps, lat 158.168 ms stddev 168.706, 0 failed
progress: 30.0 s, 316.4 tps, lat 158.239 ms stddev 188.977, 0 failed
progress: 40.0 s, 316.2 tps, lat 156.866 ms stddev 205.853, 0 failed
progress: 50.0 s, 298.1 tps, lat 170.225 ms stddev 221.545, 0 failed
progress: 60.0 s, 288.3 tps, lat 171.578 ms stddev 187.608, 0 failed
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 50
number of threads: 2
maximum number of tries: 1
duration: 60 s
number of transactions actually processed: 18332
number of failed transactions: 0 (0.000%)
latency average = 163.737 ms
latency stddev = 192.493 ms
initial connection time = 55.573 ms
tps = 304.914443 (without initial connection time)
```
Видим, что чекпоинты очень сильно влияют на производительность постгре.
При каждой "донастройке" был прирост в примерно 10%.