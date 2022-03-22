# DZ_03

Установка и настройка PostgreSQL

Цель:
создавать дополнительный диск для уже существующей виртуальной машины, размечать его и делать на нем файловую систему
переносить содержимое базы данных PostgreSQL на дополнительный диск
переносить содержимое БД PostgreSQL между виртуальными машинами

Описание/Пошаговая инструкция выполнения домашнего задания:
создайте виртуальную машину c Ubuntu 20.04 LTS (bionic) в GCE типа e2-medium в default VPC в любом регионе и зоне, например us-central1-a
Выполнено -- Пересоздал postgres@postgres-22032022 gcloud compute instances create postgres-22032022

поставьте на нее PostgreSQL 14 через sudo apt
Выполнено --  sudo apt update && sudo apt upgrade -y && sudo sh -c 'echo "deb http://apt.postgresql.org/pub/repos/apt $(lsb_release -cs)-pgdg main" > /etc/apt/sources.list.d/pgdg.list' && wget --quiet -O - https://www.postgresql.org/media/keys/ACCC4CF8.asc | sudo apt-key add - && sudo apt-get update && sudo apt-get -y install postgresql-14

sudo apt-get -y install postgresql

проверьте что кластер запущен через sudo -u postgres pg_lsclusters
Выполнено --  Ver Cluster Port Status Owner    Data directory              Log file
14  main    5432 online postgres /var/lib/postgresql/14/main /var/log/postgresql/postgresql-14-main.log

зайдите из под пользователя postgres в psql и сделайте произвольную таблицу с произвольным содержимым postgres=# create table test(c1 text); postgres=# insert into test values('1'); \q
Выполнено --
postgres@postgres-22032022:/home/anadyrov$ psql 
psql (14.2 (Ubuntu 14.2-1.pgdg20.04+1))
Type "help" for help.

postgres=# CREATE DATABASE otus2203;
CREATE DATABASE
postgres=# \q
postgres@postgres-22032022:/home/anadyrov$ psql otus2203
psql (14.2 (Ubuntu 14.2-1.pgdg20.04+1))
Type "help" for help.

otus2203=# create table test(c1 text);
CREATE TABLE
otus2203=# insert into test values('1'); \q
INSERT 0 1
otus2203=# select * from test;
 c1 
----
 1
(1 row)

остановите postgres например через sudo -u postgres pg_ctlcluster 14 main stop
Выполнено --
root@postgres-22032022:/home/anadyrov# sudo -u postgres pg_ctlcluster 14 main stop
Warning: stopping the cluster using pg_ctlcluster will mark the systemd unit as failed. Consider using systemctl:
  sudo systemctl stop postgresql@14-main
root@postgres-22032022:/home/anadyrov# pg_lsclusters
Ver Cluster Port Status Owner    Data directory              Log file
14  main    5432 down   postgres /var/lib/postgresql/14/main /var/log/postgresql/postgresql-14-main.log

создайте новый standard persistent диск GKE через Compute Engine -> Disks в том же регионе и зоне что GCE инстанс размером например 10GB
Выполнено --
gcloud compute resource-policies create snapshot-schedule default-schedule-1 --project=melodic-realm-343102 --region=us-central1 --max-retention-days=14 --on-source-disk-delete=keep-auto-snapshots --daily-schedule --start-time=01:00

добавьте свеже-созданный диск к виртуальной машине - надо зайти в режим ее редактирования и дальше выбрать пункт attach existing disk
Выполнено --

проинициализируйте диск согласно инструкции и подмонтировать файловую систему, только не забывайте менять имя диска на актуальное, в вашем случае это скорее всего будет /dev/sdb - https://www.digitalocean.com/community/tutorials/how-to-partition-and-format-storage-devices-in-linux
Выполнено --

сделайте пользователя postgres владельцем /mnt/data - chown -R postgres:postgres /mnt/data/
Выполнено --

root@postgres-22032022:/home/anadyrov# df -h
Filesystem      Size  Used Avail Use% Mounted on
/dev/root       9.6G  2.1G  7.5G  22% /
devtmpfs        2.0G     0  2.0G   0% /dev
tmpfs           2.0G   28K  2.0G   1% /dev/shm
tmpfs           393M  948K  392M   1% /run
tmpfs           5.0M     0  5.0M   0% /run/lock
tmpfs           2.0G     0  2.0G   0% /sys/fs/cgroup
/dev/loop0       56M   56M     0 100% /snap/core18/2284
/dev/loop1       62M   62M     0 100% /snap/core20/1361
/dev/loop2      256M  256M     0 100% /snap/google-cloud-sdk/228
/dev/loop3       68M   68M     0 100% /snap/lxd/22526
/dev/loop4       44M   44M     0 100% /snap/snapd/14978
/dev/sda15      105M  5.2M  100M   5% /boot/efi
tmpfs           393M     0  393M   0% /run/user/1001
/dev/sdb        9.8G   37M  9.3G   1% /mnt/data

перенесите содержимое /var/lib/postgres/14 в /mnt/data - mv /var/lib/postgresql/14 /mnt/data
Выполнено --

попытайтесь запустить кластер - sudo -u postgres pg_ctlcluster 14 main start
Выполнено --
root@postgres-22032022:/home/anadyrov# sudo -u postgres pg_ctlcluster 14 main start
Error: /var/lib/postgresql/14/main is not accessible or does not exist

напишите получилось или нет и почему
Кластер пытается запутиться с каталогом данных по умолчанию, т.к мы его перенесли необходимо прописать новый путь в файле параметров 
/etc/postgresql/14/main/postgresql.conf

задание: найти конфигурационный параметр в файлах раположенных в /etc/postgresql/10/main который надо поменять и поменяйте его - /etc/postgresql/14/main/postgresql.conf
напишите что и почему поменяли
/etc/postgresql/14/main/postgresql.conf
"data_directory = '/mnt/data/14/main'"

Поменял новый путь, запустил 
sudo -u postgres pg_ctlcluster 14 main start
root@postgres-22032022:/mnt/data/14/main# pg_lsclusters
Ver Cluster Port Status Owner    Data directory    Log file
14  main    5432 online postgres /mnt/data/14/main /var/log/postgresql/postgresql-14-main.log

зайдите через через psql и проверьте содержимое ранее созданной таблицы
postgres@postgres-22032022:/mnt/data/14/main$ psql otus2203
psql (14.2 (Ubuntu 14.2-1.pgdg20.04+1))
Type "help" for help.

otus2203=# select * from test;
 c1 
----
 1
(1 row)

задание со звездочкой *: не удаляя существующий GCE инстанс сделайте новый, поставьте на его PostgreSQL, удалите файлы с данными из /var/lib/postgres, перемонтируйте внешний диск который сделали ранее от первой виртуальной машины ко второй и запустите PostgreSQL на второй машине так чтобы он работал с данными на внешнем диске, расскажите как вы это сделали и что в итоге получилось.
Критерии оценки:
Выполнение ДЗ: 10 баллов

Почему то не получилось отцепить диск, решил склонировать и добавить в новый инстанц postgres-22032022-2
gcloud compute disks create disk-2204 --project=melodic-realm-343102 --type=pd-ssd --size=SIZE --resource-policies=projects/melodic-realm-343102/regions/us-central1/resourcePolicies/default-schedule-1 --region=us-central1 --replica-zones=projects/melodic-realm-343102/zones/us-central1-a,projects/melodic-realm-343102/zones/us-central1-c

root@postgres-22032022-2:/home/anadyrov# lsblk
NAME    MAJ:MIN RM   SIZE RO TYPE MOUNTPOINT
loop0     7:0    0  55.5M  1 loop /snap/core18/2284
loop1     7:1    0  61.9M  1 loop /snap/core20/1361
loop2     7:2    0  67.9M  1 loop /snap/lxd/22526
loop3     7:3    0 255.1M  1 loop /snap/google-cloud-sdk/228
loop4     7:4    0  43.6M  1 loop /snap/snapd/14978
sda       8:0    0    10G  0 disk 
├─sda1    8:1    0   9.9G  0 part /
├─sda14   8:14   0     4M  0 part 
└─sda15   8:15   0   106M  0 part /boot/efi
sdb       8:16   0    10G  0 disk 

root@postgres-22032022-2:/home/anadyrov# mkdir /mnt/data
root@postgres-22032022-2:/home/anadyrov# mount -o defaults /dev/sdb /mnt/data/
root@postgres-22032022-2:/home/anadyrov# df -h
Filesystem      Size  Used Avail Use% Mounted on
/dev/root       9.6G  1.7G  7.9G  18% /
devtmpfs        2.0G     0  2.0G   0% /dev
tmpfs           2.0G     0  2.0G   0% /dev/shm
tmpfs           393M  932K  392M   1% /run
tmpfs           5.0M     0  5.0M   0% /run/lock
tmpfs           2.0G     0  2.0G   0% /sys/fs/cgroup
/dev/loop0       56M   56M     0 100% /snap/core18/2284
/dev/loop1       62M   62M     0 100% /snap/core20/1361
/dev/loop2       68M   68M     0 100% /snap/lxd/22526
/dev/loop3      256M  256M     0 100% /snap/google-cloud-sdk/228
/dev/loop4       44M   44M     0 100% /snap/snapd/14978
/dev/sda15      105M  5.2M  100M   5% /boot/efi
tmpfs           393M     0  393M   0% /run/user/1001
/dev/sdb        9.8G   86M  9.2G   1% /mnt/data

Поставил postgres после удалил дефолтный каталог.

root@postgres-22032022-2:/mnt/data/14/main# sudo -u postgres pg_lsclusters
Ver Cluster Port Status Owner    Data directory              Log file
14  main    5432 online postgres /var/lib/postgresql/14/main /var/log/postgresql/postgresql-14-main.log
root@postgres-22032022-2:/mnt/data/14/main# sudo -u postgres pg_ctlcluster 14 main stop
Warning: stopping the cluster using pg_ctlcluster will mark the systemd unit as failed. Consider using systemctl:
  sudo systemctl stop postgresql@14-main
  
root@postgres-22032022-2:/mnt/data/14/main# sudo -u postgres pg_lsclusters
Ver Cluster Port Status Owner    Data directory              Log file
14  main    5432 down   postgres /var/lib/postgresql/14/main /var/log/postgresql/postgresql-14-main.log
root@postgres-22032022-2:/mnt/data/14/main# rm -rf /var/lib/postgresql

Посменял путь к склонированному диску
root@postgres-22032022-2:/mnt/data/14/main# nano /etc/postgresql/14/main/postgresql.conf
root@postgres-22032022-2:/mnt/data/14/main# chown -R postgres:postgres /mnt/data/

Запустил и проверил
root@postgres-22032022-2:/mnt/data/14/main# sudo -u postgres pg_ctlcluster 14 main start
Warning: the cluster will not be running as a systemd service. Consider using systemctl:
  sudo systemctl start postgresql@14-main
root@postgres-22032022-2:/mnt/data/14/main# sudo -u postgres pg_lsclusters
Ver Cluster Port Status Owner    Data directory    Log file
14  main    5432 online postgres /mnt/data/14/main /var/log/postgresql/postgresql-14-main.log

Подключился к базе

root@postgres-22032022-2:/mnt/data/14/main# su postgres
postgres@postgres-22032022-2:/mnt/data/14/main$ psql 
psql (14.2 (Ubuntu 14.2-1.pgdg20.04+1))
Type "help" for help.

postgres=# \l
                              List of databases
   Name    |  Owner   | Encoding | Collate |  Ctype  |   Access privileges   
-----------+----------+----------+---------+---------+-----------------------
 otus2203  | postgres | UTF8     | C.UTF-8 | C.UTF-8 | 
 postgres  | postgres | UTF8     | C.UTF-8 | C.UTF-8 | 
 template0 | postgres | UTF8     | C.UTF-8 | C.UTF-8 | =c/postgres          +
           |          |          |         |         | postgres=CTc/postgres
 template1 | postgres | UTF8     | C.UTF-8 | C.UTF-8 | =c/postgres          +
           |          |          |         |         | postgres=CTc/postgres
(4 rows)

postgres=# \q

postgres@postgres-22032022-2:/mnt/data/14/main$ psql otus2203
psql (14.2 (Ubuntu 14.2-1.pgdg20.04+1))
Type "help" for help.

otus2203=# select * from test;
 c1 
----
 1
(1 row)

otus2203=# 

Данные на месте.
