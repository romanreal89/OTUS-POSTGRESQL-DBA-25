## Установка и настройка PostgteSQL в контейнере Docker  
### Установка Docker

Будем использовать Docker Desktop Windows на базe Hyper-V.
Устанавливаем с оф сайта, убираем флажок использовать WSL.

### Запуск контейнера

```
docker volume create pgdata
docker run -d `
  --name my-postgres `
  -e POSTGRES_USER=user `
  -e POSTGRES_PASSWORD=123 `
  -e POSTGRES_DB=mydb `
  -p 54321:5432 `
  -v "pgdata:/var/lib/postgresql" `
  postgres:18
```

Вместо монтирования прямой директории был выбран docker volume.
В контейнерах начиная с версии 18+ имеются ограничения, нужно иметь в виду.

```
Error: in 18+, these Docker images are configured to store database data in a
       format which is compatible with "pg_ctlcluster" (specifically, using
       major-version-specific directory names).  This better reflects how
       PostgreSQL itself works, and how upgrades are to be performed.

       See also https://github.com/docker-library/postgres/pull/1259⁠

       Counter to that, there appears to be PostgreSQL data in:
         /var/lib/postgresql/data (unused mount/volume)

       This is usually the result of upgrading the Docker image without
       upgrading the underlying database using "pg_upgrade" (which requires both
       versions).

       The suggested container configuration for 18+ is to place a single mount
       at /var/lib/postgresql which will then place PostgreSQL data in a
       subdirectory, allowing usage of "pg_upgrade --link" without mount point
       boundary issues.

       See https://github.com/docker-library/postgres/issues/37⁠ for a (long)
       discussion around this process, and suggestions for how to do so.
```

Заходим в консоль нашего контейнера в Docker Desktop.
Подвключаемся к бд и создаем данные.

```
psql -h localhost -p 5432 -U user -d mydb

create table test(id serial, txt_col text);
CREATE TABLE

insert into test(txt_col) values('text1'),('text2'),('text3'); 
commit;
INSERT 0 1
INSERT 0 1
INSERT 0 1


select * from test;
 id | txt_col
----+------------
  1 | text1       
  2 | text2       
  3 | text3       
(3 rows)

```

Здесь внутри можно подключаться по привычному стандартному порту.
Если же мы подключаемся из pgAdmin с хоста, то нужно использовать порт 54321.

Заходим в powershell , удаляем контейнер , а затем создаем заново.
```
PS C:\WINDOWS\system32> docker stop my-postgres
my-postgres
PS C:\WINDOWS\system32> docker rm my-postgres
my-postgres
PS C:\WINDOWS\system32> docker run -d `
>>   --name my-postgres `
>>   -e POSTGRES_USER=user `
>>   -e POSTGRES_PASSWORD=123 `
>>   -e POSTGRES_DB=mydb `
>>   -p 54321:5432 `
>>   -v "pgvol:/var/lib/postgresql" `
>>   postgres:18
9152809a494c0928c709be5900b3a023d4b4efe91701bbcc5320b959e64a3ff1
```

Подключаемся изнутри докера и проверяем, что наши данные на месте.

```
psql -h localhost -p 5432 -U user -d mydb

select * from test;
 id | txt_col
----+------------
  1 | text1       
  2 | text2       
  3 | text3       
(3 rows)

```

Также проверяем, что эти же данные доступны из pgAdmin с хоста.