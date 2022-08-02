# Предварительные условия
Для выполнения данной работы используется две ВМ в Hyper-V с установленными на ними Ubuntu 20.04.  
Диск предварительно подключен к одной из ВМ для создания ФС по шине SCSI.  
Шаг установки PostgreSQL на обе ВМ пропущен.
# Инициализация базы
На **VM-1** создадим базу и внесем три тестовые строки:
```sql
$ sudo -u postgres psql

postgres=# create table test(c1 text);
CREATE TABLE
postgres=# insert into test values('1');
INSERT 0 1
postgres=# insert into test values('2');
INSERT 0 1
postgres=# insert into test values('3');
INSERT 0 1
postgres=# \q
```
Убедимся что кластер работает (не знаю зачем, мы же только что туда данные залили).  
На **VM-1**:
```
$ pg_lsclusters
Ver Cluster Port Status Owner    Data directory              Log file
14  main    5434 online postgres /var/lib/postgresql/14/main /var/log/postgresql/postgresql-14-main.log
```
Далее на той же ВМ примонтируем созданный и размеченный диск перенесем на него данные и попробуем запустить:
```bash
$ sudo mkdir /mnt/pgsql
$ sudo chown postgres:postgres /mnt/pgsql/
$ sudo systemctl stop postgresql@14-main
$ sudo mount /dev/sdb1 /mnt/pgsql/
$ sudo mv /var/lib/postgresql/14/ /mnt/pgsql/
$ sudo systemctl start postgresql@14-main.service
Job for postgresql@14-main.service failed because the service did not take the steps required by its unit configuration.
See "systemctl status postgresql@14-main.service" and "journalctl -xe" for details.
```
Попытка запуска привела к ошибке. В журнале можно увидеть точную причину возникноваения ошибки: `postgresql@14-main[3453]: Error: /var/lib/postgresql/14/main is not accessible or does not exist`.
За расположение папки с данными отвечает параметр `data_directory`, находящийся в основном конфигурационном файле. Изначально он имеет значение `/var/lib/postgresql/<cluster_version>/<cluster_name>`, так как мы полностью переместили папку версии, установим новое значение равное `/mnt/pgsql/14/main`.  
Теперь запустим кластер и проверим что имеем дело именно с нашим прежним кластером:
```bash
$ sudo systemctl start postgresql@14-main.service
$ sudo -u postgres psql -c "select * from test;"
 c1
----
 1
 2
 3
(3 rows)
```
# Подключение к другой ВМ
Отключим имеющийся диск от **VM-1** через диспетчер Hyper-V.  
После отключения служба PostgreSQL аварийно завершится из-за отсутствия файлов.  
На **VM-2** остановим и удалим папку с текущим кластером:
```bash
$ sudo pg_ctlcluster 14 main stop
$ sudo rm -rf /var/lib/postgres/14
```
Далее подключаем диск к ВМ, монтируем его и изменяем владельца. Проверяем что все нормально и наши данные на месте.
>Смена владельца необходима по той причине, что информация о нем хранится в inode по id пользователя и группы, а не по имени и могут отличаться от системы к системе. Проверить это можно выполнив команду `id -u postgres`
```bash
$ sudo mount /dev/sdb1 /var/lib/postgresql/
$ sudo chown postgres:postgres /var/lib/postgresql/ -R
$ sudo systemctl start postgresql@14-main
$ sudo -u postgres psql -c "select * from test;"
 c1
----
 1
 2
 3
(3 rows)```