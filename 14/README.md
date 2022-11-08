# Подготовка окружения
Из-за того, что все выполняется на встроенном Hyper-V, я буду использовать несколько отдельных кластеров PostgreSQL вместо отдельных VM, за исключением физической репликации, она будет выполняться с отдельной ВМ.  

Создадим и настроим три отдельных кластера на сервере:
```bash
# Создание кластера
sudo pg_createcluster 14 c1
sudo pg_createcluster 14 c2
sudo pg_createcluster 14 c3

# Включение прослушивания TCP/IP сокета и настройка WAL
sudo touch /etc/postgresql/14/c{1,2,3}/conf.d/replica.conf
cat << EOF | sudo tee /etc/postgresql/14/*/conf.d/replica.conf > /dev/null 
listen_addresses = '*'
wal_level = logical
EOF

# Разрешение на подключение
echo "host all replicator $(hostname -I | awk '{print $1}')/32 scram-sha-256" | sudo tee -a /etc/postgresql/14/c{1,2,3}/pg_hba.conf > /dev/null

# Запускаем PostgresSQL
sudo systemctl start postgresql@14-c{1,2,3}

# Создаем БД и таблицы:
sudo -u postgres psql -p 5432 -c "CREATE DATABASE c1;"
sudo -u postgres psql -p 5433 -c "CREATE DATABASE c2;"
sudo -u postgres psql -p 5434 -c "CREATE DATABASE c3;"
sudo -u postgres psql -p 5432 -d c1 -c "create table c1t1 (id int, txt text); create table c2t2(id int, txt text);"
sudo -u postgres psql -p 5433 -d c2 -c "create table c1t1 (id int, txt text); create table c2t2(id int, txt text);"
sudo -u postgres psql -p 5434 -d c3 -c "create table c1t1 (id int, txt text); create table c2t2(id int, txt text);"

# Создаем пользователя для репликации
sudo -u postgres psql -p 5432 -c "CREATE USER replicator WITH SUPERUSER PASSWORD 'Repl1c$';"
sudo -u postgres psql -p 5433 -c "CREATE USER replicator WITH SUPERUSER PASSWORD 'Repl1c$';"
sudo -u postgres psql -p 5434 -c "CREATE USER replicator WITH SUPERUSER PASSWORD 'Repl1c$';"
```

# Логическая репликация
Далее создадим необходимые публикации и подписки, проверим корректность их работы:
```pgsql
-- В кластере c1
CREATE PUBLICATION pub_c1t1 for table c1t1;
-- В кластере c2
CREATE PUBLICATION pub_c2t2 for table c2t2;

-- В кластере c1
CREATE SUBSCRIPTION sub_c2t2 CONNECTION 'host=192.168.1.203 port=5433 user=replicator password=Repl1c$ dbname=c2' publication pub_c2t2 with (copy_data = true);
-- В кластере c2
CREATE SUBSCRIPTION sub_c1t1 CONNECTION 'host=192.168.1.203 port=5432 user=replicator password=Repl1c$ dbname=c1' publication pub_c1t1 with (copy_data = true);

-- В кластере c1
INSERT INTO c1t1 (id, txt) VALUES (1, 'one'), (2, 'two');
-- В кластере c2
INSERT INTO c2t2 (id, txt) VALUES (3, 'three'), (4, 'four');
-- В кластере c1
SELECT id, txt from c2t2;
-- В кластере c2
SELECT id, txt from c1t1;
c1=# SELECT id, txt from c2t2;
 id |  txt
----+-------
  3 | three
  4 | four
(2 rows)
c2=# SELECT id, txt from c1t1;
 id | txt
----+-----
  1 | one
  2 | two
(2 rows)

-- В кластере c3
CREATE SUBSCRIPTION sub_c3_c2t2 CONNECTION 'host=192.168.1.203 port=5433 user=replicator password=Repl1c$ dbname=c2' publication pub_c2t2 with (copy_data = true);
CREATE SUBSCRIPTION sub_c3_c1t1 CONNECTION 'host=192.168.1.203 port=5432 user=replicator password=Repl1c$ dbname=c1' publication pub_c1t1 with (copy_data = true);
-- Проверим работу репликации
c3=# select * from c1t1;
 id | txt
----+-----
  1 | one
  2 | two
(2 rows)
c3=# select * from c2t2;
 id |  txt
----+-------
  3 | three
  4 | four
(2 rows)
```
# Физическая репликация
Создадим новый кластер и разрешим подключение на имеющемся для выполнения репликации.
```bash
# Создадим новый кластер replica
sudo pg_createcluster 14 replica
# Разрешим подключение на кластер c3 для физической репликации
echo "host replication replicator $(hostname -I | awk '{print $1}')/32 scram-sha-256" | sudo tee -a /etc/postgresql/14/c3/pg_hba.conf > /dev/null
echo "host replication replicator $(hostname -I | awk '{print $1}')/32 scram-sha-256" | sudo tee -a /etc/postgresql/14/replica/pg_hba.conf > /dev/null
cat << EOF | sudo tee /etc/postgresql/14/c3/conf.d/replica.conf > /dev/null 
listen_addresses = '*'
wal_level = replica
EOF
sudo systemctl restart postgresql@14-c3
```
Создадим физическую репликацию и запустим кластер
```bash
sudo rm -rf /var/lib/postgresql/14/replica/
sudo -u postgres pg_basebackup -p 5434 -R -D /var/lib/postgresql/14/replica
sudo systemctl start postgresql@14-replica.service
```
Проверим работу репликации
```
sudo -u postgres psql -p 5435
\l
                                  List of databases
   Name    |  Owner   | Encoding |   Collate   |    Ctype    |   Access privileges
-----------+----------+----------+-------------+-------------+-----------------------
 c3        | postgres | UTF8     | en_US.UTF-8 | en_US.UTF-8 |
 postgres  | postgres | UTF8     | en_US.UTF-8 | en_US.UTF-8 |
 template0 | postgres | UTF8     | en_US.UTF-8 | en_US.UTF-8 | =c/postgres          +
           |          |          |             |             | postgres=CTc/postgres
 template1 | postgres | UTF8     | en_US.UTF-8 | en_US.UTF-8 | =c/postgres          +
           |          |          |             |             | postgres=CTc/postgres
(4 rows)

postgres=# \c c3
You are now connected to database "c3" as user "postgres".
c3=# \dt
        List of relations
 Schema | Name | Type  |  Owner
--------+------+-------+----------
 public | c1t1 | table | postgres
 public | c2t2 | table | postgres
(2 rows)

-- Создадим к c3 новую таблицу
create table test_table (id int);

-- Эта же база сразу появилась и на реплике
c3=# \dt
           List of relations
 Schema |    Name    | Type  |  Owner
--------+------------+-------+----------
 public | c1t1       | table | postgres
 public | c2t2       | table | postgres
 public | test_table | table | postgres
(3 rows)