##Установка и настройка PostgteSQL + Уровни изоляций транзакций в PostgreSQL
### Установка PostgreSQL

Для начала установим постгре на ВМ UBUNTU 24.04

Заходим на страницу установки на оф сайте

https://www.postgresql.org/download/linux/ubuntu/

т.к. нам нужен последний дистрибутив, выбирай 2й вариант установки

```

root@otus-PC:/home/otus# sudo apt install -y postgresql-common
sudo /usr/share/postgresql-common/pgdg/apt.postgresql.org.sh
Reading package lists... Done
Building dependency tree... Done
Reading state information... Done
The following package was automatically installed and is no longer required:
&nbsp; libllvm19
Use 'sudo apt autoremove' to remove it.
The following additional packages will be installed:
&nbsp; libcommon-sense-perl libjson-perl libjson-xs-perl libtypes-serialiser-perl postgresql-client-common
The following NEW packages will be installed:
&nbsp; libcommon-sense-perl libjson-perl libjson-xs-perl libtypes-serialiser-perl postgresql-client-common postgresql-common
0 upgraded, 6 newly installed, 0 to remove and 0 not upgraded.
Need to get 395 kB of archives.
After this operation, 1,326 kB of additional disk space will be used.
Get:1 http://archive.ubuntu.com/ubuntu noble/main amd64 libjson-perl all 4.10000-1 \[81.9 kB]
Get:2 http://archive.ubuntu.com/ubuntu noble-updates/main amd64 postgresql-client-common all 257build1.1 \[36.4 kB]
Get:3 http://archive.ubuntu.com/ubuntu noble-updates/main amd64 postgresql-common all 257build1.1 \[161 kB]
Get:4 http://archive.ubuntu.com/ubuntu noble/main amd64 libcommon-sense-perl amd64 3.75-3build3 \[20.4 kB]
Get:5 http://archive.ubuntu.com/ubuntu noble/main amd64 libtypes-serialiser-perl all 1.01-1 \[11.6 kB]
Get:6 http://archive.ubuntu.com/ubuntu noble-updates/main amd64 libjson-xs-perl amd64 4.040-0ubuntu0.24.04.1 \[83.7 kB]
Fetched 395 kB in 0s (816 kB/s)
Preconfiguring packages ...
Selecting previously unselected package libjson-perl.
(Reading database ... 192643 files and directories currently installed.)
Preparing to unpack .../0-libjson-perl\_4.10000-1\_all.deb ...
Unpacking libjson-perl (4.10000-1) ...
Selecting previously unselected package postgresql-client-common.
Preparing to unpack .../1-postgresql-client-common\_257build1.1\_all.deb ...
Unpacking postgresql-client-common (257build1.1) ...
Selecting previously unselected package postgresql-common.
Preparing to unpack .../2-postgresql-common\_257build1.1\_all.deb ...
Adding 'diversion of /usr/bin/pg\_config to /usr/bin/pg\_config.libpq-dev by postgresql-common'
Unpacking postgresql-common (257build1.1) ...
Selecting previously unselected package libcommon-sense-perl:amd64.
Preparing to unpack .../3-libcommon-sense-perl\_3.75-3build3\_amd64.deb ...
Unpacking libcommon-sense-perl:amd64 (3.75-3build3) ...
Selecting previously unselected package libtypes-serialiser-perl.
Preparing to unpack .../4-libtypes-serialiser-perl\_1.01-1\_all.deb ...
Unpacking libtypes-serialiser-perl (1.01-1) ...
Selecting previously unselected package libjson-xs-perl.
Preparing to unpack .../5-libjson-xs-perl\_4.040-0ubuntu0.24.04.1\_amd64.deb ...
Unpacking libjson-xs-perl (4.040-0ubuntu0.24.04.1) ...
Setting up postgresql-client-common (257build1.1) ...
Setting up libcommon-sense-perl:amd64 (3.75-3build3) ...
Setting up libtypes-serialiser-perl (1.01-1) ...
Setting up libjson-perl (4.10000-1) ...
Setting up libjson-xs-perl (4.040-0ubuntu0.24.04.1) ...
Setting up postgresql-common (257build1.1) ...


Creating config file /etc/postgresql-common/createcluster.conf with new version
Building PostgreSQL dictionaries from installed myspell/hunspell packages...
&nbsp; en\_us
Removing obsolete dictionary files:
Created symlink /etc/systemd/system/multi-user.target.wants/postgresql.service → /usr/lib/systemd/system/postgresql.service.
Processing triggers for man-db (2.12.0-4build2) ...
This script will enable the PostgreSQL APT repository on apt.postgresql.org on
your system. The distribution codename used will be noble-pgdg.


Press Enter to continue, or Ctrl-C to abort.


Using keyring /usr/share/postgresql-common/pgdg/apt.postgresql.org.gpg
Writing /etc/apt/sources.list.d/pgdg.sources ...

Running apt-get update ...
Hit:1 http://archive.ubuntu.com/ubuntu noble InRelease
Hit:2 http://security.ubuntu.com/ubuntu noble-security InRelease
Hit:3 http://archive.ubuntu.com/ubuntu noble-updates InRelease
Hit:4 http://archive.ubuntu.com/ubuntu noble-backports InRelease
Get:5 https://apt.postgresql.org/pub/repos/apt noble-pgdg InRelease \[107 kB]
Get:6 https://apt.postgresql.org/pub/repos/apt noble-pgdg/main amd64 Packages \[353 kB]
Fetched 460 kB in 1s (881 kB/s)
Reading package lists... Done

You can now start installing packages from apt.postgresql.org.

Have a look at https://wiki.postgresql.org/wiki/Apt for more information;
most notably the FAQ at https://wiki.postgresql.org/wiki/Apt/FAQ
root@otus-PC:/home/otus#
```

Теперь установим 18ю версию

```
root@otus-PC:/home/otus# apt install postgresql-18
Reading package lists... Done
Building dependency tree... Done
Reading state information... Done
The following additional packages will be installed:
&nbsp; libpq5 liburing2 postgresql-18-jit postgresql-client-18 postgresql-client-common postgresql-common
Suggested packages:
&nbsp; libpq-oauth postgresql-doc-18
The following NEW packages will be installed:
&nbsp; libpq5 liburing2 postgresql-18 postgresql-18-jit postgresql-client-18
The following packages will be upgraded:
&nbsp; postgresql-client-common postgresql-common
2 upgraded, 5 newly installed, 0 to remove and 0 not upgraded.
Need to get 19.9 MB of archives.
After this operation, 72.5 MB of additional disk space will be used.
Do you want to continue? \[Y/n]


Get:1 http://archive.ubuntu.com/ubuntu noble/main amd64 liburing2 amd64 2.5-1build1 \[21.1 kB]
Get:2 https://apt.postgresql.org/pub/repos/apt noble-pgdg/main amd64 postgresql-common all 287.pgdg24.04+1 \[112 kB]
Get:3 https://apt.postgresql.org/pub/repos/apt noble-pgdg/main amd64 postgresql-client-common all 287.pgdg24.04+1 \[47.9 kB]
Get:4 https://apt.postgresql.org/pub/repos/apt noble-pgdg/main amd64 libpq5 amd64 18.1-1.pgdg24.04+2 \[245 kB]
Get:5 https://apt.postgresql.org/pub/repos/apt noble-pgdg/main amd64 postgresql-client-18 amd64 18.1-1.pgdg24.04+2 \[2,086 kB]
Get:6 https://apt.postgresql.org/pub/repos/apt noble-pgdg/main amd64 postgresql-18 amd64 18.1-1.pgdg24.04+2 \[7,516 kB]
Get:7 https://apt.postgresql.org/pub/repos/apt noble-pgdg/main amd64 postgresql-18-jit amd64 18.1-1.pgdg24.04+2 \[9,871 kB]
Fetched 19.9 MB in 2s (13.1 MB/s)
Preconfiguring packages ...
(Reading database ... 192864 files and directories currently installed.)
Preparing to unpack .../0-postgresql-common\_287.pgdg24.04+1\_all.deb ...
Leaving 'diversion of /usr/bin/pg\_config to /usr/bin/pg\_config.libpq-dev by postgresql-common'
Unpacking postgresql-common (287.pgdg24.04+1) over (257build1.1) ...
Preparing to unpack .../1-postgresql-client-common\_287.pgdg24.04+1\_all.deb ...
Unpacking postgresql-client-common (287.pgdg24.04+1) over (257build1.1) ...
Selecting previously unselected package libpq5:amd64.
Preparing to unpack .../2-libpq5\_18.1-1.pgdg24.04+2\_amd64.deb ...
Unpacking libpq5:amd64 (18.1-1.pgdg24.04+2) ...
Selecting previously unselected package liburing2:amd64.
Preparing to unpack .../3-liburing2\_2.5-1build1\_amd64.deb ...
Unpacking liburing2:amd64 (2.5-1build1) ...
Selecting previously unselected package postgresql-client-18.
Preparing to unpack .../4-postgresql-client-18\_18.1-1.pgdg24.04+2\_amd64.deb ...
Unpacking postgresql-client-18 (18.1-1.pgdg24.04+2) ...
Selecting previously unselected package postgresql-18.
Preparing to unpack .../5-postgresql-18\_18.1-1.pgdg24.04+2\_amd64.deb ...
Unpacking postgresql-18 (18.1-1.pgdg24.04+2) ...
Selecting previously unselected package postgresql-18-jit.
Preparing to unpack .../6-postgresql-18-jit\_18.1-1.pgdg24.04+2\_amd64.deb ...
Unpacking postgresql-18-jit (18.1-1.pgdg24.04+2) ...
Setting up postgresql-client-common (287.pgdg24.04+1) ...
Removing obsolete conffile /etc/postgresql-common/supported\_versions ...
Setting up libpq5:amd64 (18.1-1.pgdg24.04+2) ...
Setting up postgresql-common (287.pgdg24.04+1) ...
Installing new version of config file /etc/postgresql-common/pg\_upgradecluster.d/analyze ...
Replacing config file /etc/postgresql-common/createcluster.conf with new version
Setting up liburing2:amd64 (2.5-1build1) ...
Setting up postgresql-client-18 (18.1-1.pgdg24.04+2) ...
update-alternatives: using /usr/share/postgresql/18/man/man1/psql.1.gz to provide /usr/share/man/man1/psql.1.gz (psql.1.gz) in auto mode
Setting up postgresql-18 (18.1-1.pgdg24.04+2) ...
Creating new PostgreSQL cluster 18/main ...
/usr/lib/postgresql/18/bin/initdb -D /var/lib/postgresql/18/main --auth-local peer --auth-host scram-sha-256 --no-instructions
The files belonging to this database system will be owned by user "postgres".
This user must also own the server process.

The database cluster will be initialized with locale "en\_US.UTF-8".
The default database encoding has accordingly been set to "UTF8".
The default text search configuration will be set to "english".

Data page checksums are enabled.

fixing permissions on existing directory /var/lib/postgresql/18/main ... ok
creating subdirectories ... ok
selecting dynamic shared memory implementation ... posix
selecting default "max\_connections" ... 100
selecting default "shared\_buffers" ... 128MB
selecting default time zone ... Europe/Moscow
creating configuration files ... ok
running bootstrap script ... ok
performing post-bootstrap initialization ... ok
syncing data to disk ... ok
Setting up postgresql-18-jit (18.1-1.pgdg24.04+2) ...
Processing triggers for libc-bin (2.39-0ubuntu8.6) ...
Processing triggers for man-db (2.12.0-4build2) ...
root@otus-PC:/home/otus#
```

Теперь поправим файлы конфигурации

/etc/postgresql/18/main

Добавим доступ для всех, т.к. это тестовая среда внутри сети, допустим полный доступ

pg\_hba.conf
```
host    all             all             0.0.0.0/0               trust
```

Добавил возможность сетевого входа
postgresql.confg
```
listen\_addresses = '\*'  
```

Сделаем рестарт постгре и пробуем подключиться из-под любого клиента.
В моем случае это DBeaver - коннект успешен.

### Работа с уровнями изоляций



