# Вступление
Раз мы все равно показываем как круто у нас сохраняются данные в докере, добавим вариант когда у нас они НЕ сохранились.  
Файл compose из которого производится запуск контейнеров так же загружен на Github.
Также мой пользователь является членом группы `docker`, выполнение всех операций с docker возможно без повышения привилегий.
# Первое создание контейнеров и наполнение БД
> Так как все действие происходит из папочки с именем 03, либо поведение compose немного изменилось в установленной мною версии, имена контейнеров не совсем совпадают с именами указанными в compose.  

1. Создадим папку postgres в той же папке где лежит файл compose
   ```
   $ mkdir postgres
   ```

2. Выполним запуск контейнеров и посмотрим их имена
    ```
    $ docker ps --format "table {{.ID}}\t{{.Image}}\t{{.Names}}"
    CONTAINER ID   IMAGE                NAMES            PORTS
   ca317d473911   postgres:14-alpine   03_pg_bind_1     0.0.0.0:5432->5432/tcp, :::5432->5432/tcp
   59b95686e5a2   postgres:14-alpine   03_pg_nobind_1   0.0.0.0:5433->5432/tcp, :::5433->5432/tcp
   f141d7ad45ca   postgres:14-alpine   03_pg_client_1   5432/tcp
    
    ```
3. Зайдем внутрь нашего контейнера с клиентом
   ```
   docker exec -it 03_pg_client_1 bash
   ```
4. Подключимся из него поочередно к контейнерам `03_pg_bind_1` и `03_pg_nobind_1` и выполним создание и наполнение баз.  
Убедимся что подключение произошло именно к контейнеру с клиентом
   ```
   bash-5.1# su postgres
   / $ psql
   # Так же контейнеры у нас отличаются именами баз данных
   postgres=# \l
      Name    |  Owner   | Encoding |  Collate   |   Ctype    |   Access privileges
   -----------+----------+----------+------------+------------+-----------------------
    client    | postgres | UTF8     | en_US.utf8 | en_US.utf8 |
    postgres  | postgres | UTF8     | en_US.utf8 | en_US.utf8 |
    template0 | postgres | UTF8     | en_US.utf8 | en_US.utf8 | =c/postgres          +
              |          |          |            |            | postgres=CTc/postgres
    template1 | postgres | UTF8     | en_US.utf8 | en_US.utf8 | =c/postgres          +
              |          |          |            |            | postgres=CTc/postgres
   (4 rows)

   postgres=# \q
   ```
   > А если надо просто один раз быстро создать и наполнить ее небольшим количеством данных, можно использовать команду `docker exec -it 03_pg_bind_1 su -c "psql -d binded -c \"create table binded (id int, name text); insert into binded values (1, 'one'), (2, 'two'), (3, 'three'), (4, 'four'); select * from binded;\"" postgres`  
   > 
   Из клиента подключимся к 03_pg_bind_1
   ```
   bash-5.1# psql -h 03_pg_bind_1 -U postgres
   Password for user postgres:
   postgres=# \l
                                    List of databases
      Name    |  Owner   | Encoding |  Collate   |   Ctype    |   Access privileges
   -----------+----------+----------+------------+------------+-----------------------
    binded    | postgres | UTF8     | en_US.utf8 | en_US.utf8 |
    postgres  | postgres | UTF8     | en_US.utf8 | en_US.utf8 |
    template0 | postgres | UTF8     | en_US.utf8 | en_US.utf8 | =c/postgres          +
              |          |          |            |            | postgres=CTc/postgres
    template1 | postgres | UTF8     | en_US.utf8 | en_US.utf8 | =c/postgres          +
              |          |          |            |            | postgres=CTc/postgres
   (4 rows)

   postgres=# \c binded
   You are now connected to database "binded" as user "postgres".
   binded=# create table binded (id int, name text);
   CREATE TABLE
   binded=# insert into binded values (1, 'one'), (2, 'two'), (3, 'three'), (4, 'four');
   INSERT 0 4
   binded=# select * from binded;
    id | name
   ----+-------
     1 | one
     2 | two
     3 | three
     4 | four
   (4 rows)

   binded=# ^D\q
   ```
   Повторим для 03_pg_nobind_1
   ```
   bash-5.1# psql -h 03_pg_nobind_1 -U postgres
   Password for user postgres:

   postgres=# \l
                                    List of databases
      Name    |  Owner   | Encoding |  Collate   |   Ctype    |   Access privileges
   -----------+----------+----------+------------+------------+-----------------------
    nobinded  | postgres | UTF8     | en_US.utf8 | en_US.utf8 |
    postgres  | postgres | UTF8     | en_US.utf8 | en_US.utf8 |
    template0 | postgres | UTF8     | en_US.utf8 | en_US.utf8 | =c/postgres          +
              |          |          |            |            | postgres=CTc/postgres
    template1 | postgres | UTF8     | en_US.utf8 | en_US.utf8 | =c/postgres          +
              |          |          |            |            | postgres=CTc/postgres
   (4 rows)

   postgres=# \c nobinded
   You are now connected to database "nobinded" as user "postgres".
   nobinded=# create table nobinded (id int, name text);
   CREATE TABLE
   nobinded=# insert into nobinded values (1, 'one'), (2, 'two'), (3, 'three'), (4, 'four');
   INSERT 0 4
   nobinded=# select * from nobinded;
    id | name
   ----+-------
     1 | one
     2 | two
     3 | three
     4 | four
   (4 rows)

   nobinded=# ^D\q
   ```
   Проверим подключение к контейнерам с хостовой машины
   ```
   $ psql -h localhost -U postgres
   Password for user postgres:

   postgres=# \l
                                    List of databases
      Name    |  Owner   | Encoding |  Collate   |   Ctype    |   Access privileges
   -----------+----------+----------+------------+------------+-----------------------
    binded    | postgres | UTF8     | en_US.utf8 | en_US.utf8 |
    postgres  | postgres | UTF8     | en_US.utf8 | en_US.utf8 |
    template0 | postgres | UTF8     | en_US.utf8 | en_US.utf8 | =c/postgres          +
              |          |          |            |            | postgres=CTc/postgres
    template1 | postgres | UTF8     | en_US.utf8 | en_US.utf8 | =c/postgres          +
              |          |          |            |            | postgres=CTc/postgres
   (4 rows)

   postgres=# \c binded
   You are now connected to database "binded" as user "postgres".
   binded=# select * from binded ;
    id | name
   ----+-------
     1 | one
     2 | two
     3 | three
     4 | four
   (4 rows)

   binded=#\q

   $ psql -h localhost -p 5433 -U postgres
   Password for user postgres:

   postgres=# \l
                                    List of databases
      Name    |  Owner   | Encoding |  Collate   |   Ctype    |   Access privileges
   -----------+----------+----------+------------+------------+-----------------------
    nobinded  | postgres | UTF8     | en_US.utf8 | en_US.utf8 |
    postgres  | postgres | UTF8     | en_US.utf8 | en_US.utf8 |
    template0 | postgres | UTF8     | en_US.utf8 | en_US.utf8 | =c/postgres          +
              |          |          |            |            | postgres=CTc/postgres
    template1 | postgres | UTF8     | en_US.utf8 | en_US.utf8 | =c/postgres          +
              |          |          |            |            | postgres=CTc/postgres
   (4 rows)

   postgres=# \c nobinded
   You are now connected to database "nobinded" as user "postgres".
   nobinded=# select * from nobinded ;
    id | name
   ----+-------
     1 | one
     2 | two
     3 | three
     4 | four
   (4 rows)

   nobinded=# \q
   ```
   Подключиться по именам `03_pg_bind_1` и `03_pg_nobind_1` нам бы не удалось, так как существуют они внутри отдельной сети докера, нам же предоставляются порты 5432 и 5433.
# Перезапуск контейнеров и повторная проверка
Остановим и вновь запустим контейнеры
```
$ docker-compose down
$ docker-compose up -d
```
И проверим наши данные 
```
$ docker exec -it 03_pg_bind_1 su -c "psql -d binded -c \"select * from binded;\"" postgres
 id | name
----+-------
  1 | one
  2 | two
  3 | three
  4 | four
(4 rows)

$ docker exec -it 03_pg_nobind_1 su -c "psql -d nobinded -c \"select * from nobinded;\"" postgres
ERROR:  relation "nobinded" does not exist
LINE 1: select * from nobinded;
                      ^
```                      
Как мы видим, данные в контейнере у которого был добавлен биндинг папки сохранились, в то время как у контейнера без биндинга данные оказались потеряны.

# Ошибки и пути их решения
Проврка персистентности удалась только со второго раза. В первый раз, `target` у биндинга локальной папки был установлен в значние `/var/lib/postgresql`, в результате docker создавал внутри нее папку `data`, но все свои файлы сохранял в своей локальной директории. В системе присутствовало две точки монтирования:
```
/dev/mapper/ubuntu--vg-ubuntu--lv
                      39470400   8205436  29230276  22% /var/lib/postgresql
/dev/mapper/ubuntu--vg-ubuntu--lv
                      39470400   8205436  29230276  22% /var/lib/postgresql/data
```
При этом, в логах контейнера и compose никаких ошибок не было. Никакой информации подобному поведению мне найти не удалось, буду признателен если сможете мне разъяснить этот момент или предоставить ссылку где можно ознакомиться с причинами подобного поведения.