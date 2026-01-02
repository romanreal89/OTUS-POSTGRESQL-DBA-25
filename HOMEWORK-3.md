## Физичесский уровень PostgreSQL
### Создание и настройка VM

Будем использовать ВМ из предыдущих ДЗ.
ВМ создана на базе Proxmox Virtual Environment 8.4.14.

### Создание тестовых данных. Остановка кластера.

Проверяем, что кластер доступен
```
root@otus-PC:/home/otus# sudo -u postgres pg_lsclusters
Ver Cluster Port Status Owner    Data directory              Log file
18  main    5432 online postgres /var/lib/postgresql/18/main /var/log/postgresql/postgresql-18-main.log
```
Создаем тестовые данные
```
root@otus-PC:/home/otus# sudo -u postgres psql
psql (18.1 (Ubuntu 18.1-1.pgdg24.04+2))
Type "help" for help.

postgres=# create table test(c1 text);
CREATE TABLE
postgres=# insert into test values('1');
INSERT 0 1
postgres=# \q
```
Останавливаем кластер
```
root@otus-PC:/home/otus# sudo -u postgres pg_ctlcluster 18 main stop
Warning: stopping the cluster using pg_ctlcluster will mark the systemd unit as failed. Consider using systemctl:
  sudo systemctl stop postgresql@18-main
root@otus-PC:/home/otus#
```

### Добавляем диск

Добавляем диск через интерфейс проксмокса.
Затем приступаем к настройке.
Отобразим список доступных дисков
```
root@otus-PC:/home/otus# lsblk
NAME   MAJ:MIN RM   SIZE RO TYPE MOUNTPOINTS
loop0    7:0    0     4K  1 loop /snap/bare/5
loop1    7:1    0 245.1M  1 loop /snap/firefox/6565
loop2    7:2    0   516M  1 loop /snap/gnome-42-2204/202
loop3    7:3    0  73.9M  1 loop /snap/core22/2045
loop4    7:4    0  11.1M  1 loop /snap/firmware-updater/167
loop5    7:5    0  10.8M  1 loop /snap/snap-store/1270
loop6    7:6    0  49.3M  1 loop /snap/snapd/24792
loop7    7:7    0   576K  1 loop /snap/snapd-desktop-integration/315
loop8    7:8    0  91.7M  1 loop /snap/gtk-common-themes/1535
loop9    7:9    0    74M  1 loop /snap/core22/2193
loop10   7:10   0  18.5M  1 loop /snap/firmware-updater/210
loop11   7:11   0 516.2M  1 loop /snap/gnome-42-2204/226
loop12   7:12   0  50.9M  1 loop /snap/snapd/25577
loop13   7:13   0 250.8M  1 loop /snap/firefox/7559
sda      8:0    0    32G  0 disk
├─sda1   8:1    0     1M  0 part
└─sda2   8:2    0    32G  0 part /
sdb      8:16   0    10G  0 disk
sr0     11:0    1   5.9G  0 rom  /media/otus/Ubuntu 24.04.3 LTS amd64
```
Создадим раздел и файловую структуру
```
root@otus-PC:/home/otus# fdisk /dev/sdb

Welcome to fdisk (util-linux 2.39.3).
Changes will remain in memory only, until you decide to write them.
Be careful before using the write command.

Device does not contain a recognized partition table.
Created a new DOS (MBR) disklabel with disk identifier 0x360ba320.

Command (m for help): n
Partition type
   p   primary (0 primary, 0 extended, 4 free)
   e   extended (container for logical partitions)
Select (default p): p
Partition number (1-4, default 1):
First sector (2048-20971519, default 2048):
Last sector, +/-sectors or +/-size{K,M,G,T,P} (2048-20971519, default 20971519):

Created a new partition 1 of type 'Linux' and of size 10 GiB.

Command (m for help): w
The partition table has been altered.
Calling ioctl() to re-read partition table.
Syncing disks.

root@otus-PC:/home/otus# mkfs.ext4 /dev/sdb1
mke2fs 1.47.0 (5-Feb-2023)
Discarding device blocks: done
Creating filesystem with 2621184 4k blocks and 655360 inodes
Filesystem UUID: bbdcb660-af19-4ba7-a363-b11610c4d1a1
Superblock backups stored on blocks:
        32768, 98304, 163840, 229376, 294912, 819200, 884736, 1605632

Allocating group tables: done
Writing inode tables: done
Creating journal (16384 blocks): done
Writing superblocks and filesystem accounting information: done
```
Монтирование.
```
root@otus-PC:/home/otus# mkdir /mnt/data
root@otus-PC:/home/otus# mount /dev/sdb1 /mnt/data
```
Добавим запись в автозагрузку.
```
root@otus-PC:/home/otus# nano /etc/fstab

/dev/sdb1    /mnt/data    ext4    defaults    0    1

root@otus-PC:/home/otus# nano /etc/fstab
root@otus-PC:/home/otus# mount -a
root@otus-PC:/home/otus# df -h
Filesystem      Size  Used Avail Use% Mounted on
tmpfs           392M  1.6M  391M   1% /run
/dev/sda2        32G   11G   19G  37% /
tmpfs           2.0G     0  2.0G   0% /dev/shm
tmpfs           5.0M     0  5.0M   0% /run/lock
tmpfs           392M  116K  392M   1% /run/user/1000
/dev/sr0        6.0G  6.0G     0 100% /media/otus/Ubuntu 24.04.3 LTS amd64
/dev/sdb1       9.8G   24K  9.3G   1% /mnt/data
root@otus-PC:/home/otus#
```
### Перенос данных

Назначаем права. Переносим директорию. Пробуем запустить.
```
root@otus-PC:/home/otus# chown -R postgres:postgres /mnt/data/
root@otus-PC:/home/otus# mv /var/lib/postgresql/18 /mnt/data
root@otus-PC:/home/otus# sudo -u postgres pg_ctlcluster 18 main start
Error: /var/lib/postgresql/18/main is not accessible or does not exist
```
Выдало ошибку. Что вполне логично, т.к. в конфиге путь все еще старый.
Меняем путь. Запускаем и проверяем.
```
root@otus-PC:/home/otus# sudo -u postgres pg_ctlcluster 18 main start
Warning: the cluster will not be running as a systemd service. Consider using systemctl:
  sudo systemctl start postgresql@18-main
root@otus-PC:/home/otus# sudo -u postgres pg_lsclusters
Ver Cluster Port Status Owner    Data directory    Log file
18  main    5432 online postgres /mnt/data/18/main /var/log/postgresql/postgresql-18-main.log
```
Теперь все работает как нужно.
Проверяем, что наши данные на месте.
```
root@otus-PC:/home/otus# sudo -u postgres psql
psql (18.1 (Ubuntu 18.1-1.pgdg24.04+2))
Type "help" for help.

postgres=# select * from test;
 c1
----
 1
(1 row)
```
Данные на месте.