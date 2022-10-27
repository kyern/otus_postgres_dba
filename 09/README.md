# Включение логгирования блокировок
За включение логгирования блокировок в БД отвечает параметр `log_lock_waits`. При этом в лог пишутся только блокировки, время ожидания которых равно или больше значения параметра `deadlock_timeout`.  
Установим оба параметра:
```bash
cat << EOF | sudo tee /etc/postgresql/14/main/conf.d/locks.conf > /dev/null
log_lock_waits = on
deadlock_timeout = 200ms
EOF
```
И применим изменения в PostgreSQL:
```bash
sudo -u postgresql psql -c "select pg_reload_conf();"
```
# Блокировка
Для воспроизведения блокировок нам нужна хоть какая-то таблица и данные в ней. Создадим все что нам необходимо:
```sql
create table locks (id int, caption text);
insert into locks values
    (1, 'one'),
    (2, 'two'),
    (3, 'three');
```
И воспроизведем блокировку выполняя следующие команды в трех разных сеансах:
```sql
-- В первом сеансе
BEGIN;
update locks set caption = 'The Chosen One' where id = 1;
-- Во втором сеансе
BEGIN;
update locks set caption = 'Один' where id = 1;
-- В третьем сеансе
BEGIN;
update locks set caption = 'All as one!';
```
Видим в логах PostgreSQL следующие ошибки:
```
2022-10-27 04:38:31.213 UTC [59394] kyern@kyern LOG:  process 59394 still waiting for ShareLock on transaction 356059 after 200.084 ms
2022-10-27 04:38:31.213 UTC [59394] kyern@kyern DETAIL:  Process holding the lock: 59392. Wait queue: 59394.
2022-10-27 04:38:31.213 UTC [59394] kyern@kyern CONTEXT:  while updating tuple (0,1) in relation "locks"
2022-10-27 04:38:31.213 UTC [59394] kyern@kyern STATEMENT:  update locks set caption = 'Один' where id = 1;
2022-10-27 04:38:40.170 UTC [59397] kyern@kyern LOG:  process 59397 still waiting for ExclusiveLock on tuple (0,1) of relation 16631 of database 16620 after 200.093 ms
2022-10-27 04:38:40.170 UTC [59397] kyern@kyern DETAIL:  Process holding the lock: 59394. Wait queue: 59397.
2022-10-27 04:38:40.170 UTC [59397] kyern@kyern STATEMENT:  update locks set caption = 'All as one!';
```
Посмотрим все имеющиеся блокировки:
```sql
SELECT pid, locktype, relation::REGCLASS, virtualxid AS virtxid, transactionid AS xid, mode, granted, pg_blocking_pids(pid) AS wait_for FROM pg_locks order by pid;

  pid  |   locktype    | relation | virtxid |  xid   |       mode       | granted | wait_for
-------+---------------+----------+---------+--------+------------------+---------+----------
 59392 | transactionid |          |         | 356059 | ExclusiveLock    | t       | {}
 59392 | relation      | locks    |         |        | RowExclusiveLock | t       | {}
 59392 | virtualxid    |          | 3/19036 |        | ExclusiveLock    | t       | {}
 59394 | transactionid |          |         | 356060 | ExclusiveLock    | t       | {59392}
 59394 | virtualxid    |          | 4/3     |        | ExclusiveLock    | t       | {59392}
 59394 | relation      | locks    |         |        | RowExclusiveLock | t       | {59392}
 59394 | transactionid |          |         | 356059 | ShareLock        | f       | {59392}
 59394 | tuple         | locks    |         |        | ExclusiveLock    | t       | {59392}
 59397 | transactionid |          |         | 356061 | ExclusiveLock    | t       | {59394}
 59397 | relation      | locks    |         |        | RowExclusiveLock | t       | {59394}
 59397 | virtualxid    |          | 5/9     |        | ExclusiveLock    | t       | {59394}
 59397 | tuple         | locks    |         |        | ExclusiveLock    | f       | {59394}
 59697 | virtualxid    |          | 6/130   |        | ExclusiveLock    | t       | {}
 59697 | relation      | pg_locks |         |        | AccessShareLock  | t       | {}
(14 rows)
```
Как мы видим, все три транзакции запросили RowExclusiveLock, но при этом транзакции из сеансов с pid `59394` ожидают завершения `59392`, а `59397` ожидает `59394`, в то время как сам `59392` никого не ждет.  
Добавим еще один апдейт: 
```sql
BEGIN;
update locks set caption = 'The Chosen One' where id = 1;
```
СПисок блокировок обновится:
```sql
SELECT pid, locktype, relation::REGCLASS, virtualxid AS virtxid, transactionid AS xid, mode, granted, pg_blocking_pids(pid) AS wait_for FROM pg_locks order by pid;
  pid  |   locktype    | relation | virtxid |  xid   |       mode       | granted |   wait_for
-------+---------------+----------+---------+--------+------------------+---------+---------------
 59392 | relation      | locks    |         |        | RowExclusiveLock | t       | {}
 59392 | transactionid |          |         | 356059 | ExclusiveLock    | t       | {}
 59392 | virtualxid    |          | 3/19036 |        | ExclusiveLock    | t       | {}
 59394 | virtualxid    |          | 4/3     |        | ExclusiveLock    | t       | {59392}
 59394 | relation      | locks    |         |        | RowExclusiveLock | t       | {59392}
 59394 | tuple         | locks    |         |        | ExclusiveLock    | t       | {59392}
 59394 | transactionid |          |         | 356060 | ExclusiveLock    | t       | {59392}
 59394 | transactionid |          |         | 356059 | ShareLock        | f       | {59392}
 59397 | relation      | locks    |         |        | RowExclusiveLock | t       | {59394}
 59397 | virtualxid    |          | 5/9     |        | ExclusiveLock    | t       | {59394}
 59397 | tuple         | locks    |         |        | ExclusiveLock    | f       | {59394}
 59397 | transactionid |          |         | 356061 | ExclusiveLock    | t       | {59394}
 59697 | virtualxid    |          | 6/131   |        | ExclusiveLock    | t       | {}
 59697 | relation      | pg_locks |         |        | AccessShareLock  | t       | {}
 59837 | tuple         | locks    |         |        | ExclusiveLock    | f       | {59394,59397}
 59837 | virtualxid    |          | 7/68    |        | ExclusiveLock    | t       | {59394,59397}
 59837 | transactionid |          |         | 356062 | ExclusiveLock    | t       | {59394,59397}
 59837 | relation      | locks    |         |        | RowExclusiveLock | t       | {59394,59397}
(18 rows)
```
Теперь транзация из pid `59837` ждет завершения и `59394`, и `59397`.
Завершим транзакцию в первом сеансе и посмотрим как изменятся блокировки:
```sql
SELECT pid, locktype, relation::REGCLASS, virtualxid AS virtxid, transactionid AS xid, mode, granted, pg_blocking_pids(pid) AS wait_for FROM pg_locks;

  pid  |   locktype    | relation | virtxid |  xid   |       mode       | granted | wait_for
-------+---------------+----------+---------+--------+------------------+---------+----------
 59837 | relation      | locks    |         |        | RowExclusiveLock | t       | {59394}
 59837 | virtualxid    |          | 7/68    |        | ExclusiveLock    | t       | {59394}
 59697 | relation      | pg_locks |         |        | AccessShareLock  | t       | {}
 59697 | virtualxid    |          | 6/133   |        | ExclusiveLock    | t       | {}
 59397 | relation      | locks    |         |        | RowExclusiveLock | t       | {59394}
 59397 | virtualxid    |          | 5/9     |        | ExclusiveLock    | t       | {59394}
 59394 | relation      | locks    |         |        | RowExclusiveLock | t       | {}
 59394 | virtualxid    |          | 4/3     |        | ExclusiveLock    | t       | {}
 59837 | transactionid |          |         | 356060 | ShareLock        | f       | {59394}
 59394 | transactionid |          |         | 356060 | ExclusiveLock    | t       | {}
 59397 | transactionid |          |         | 356060 | ShareLock        | f       | {59394}
 59397 | transactionid |          |         | 356061 | ExclusiveLock    | t       | {59394}
 59837 | transactionid |          |         | 356062 | ExclusiveLock    | t       | {59394}
(13 rows)
```
Транзакция из сеанса `59392` завершилась и теперь `59394` может успешно завершиться, а транзакция из `59837` перестала ожидать `59397`.  
После завершения второй транзакции, третья (апдейт всей таблицы) перестает ожидать остальные транзакции и котов завершиться, а четвертая уже начинает ожидать третью и сможет сработать только после того как она звершит работу:
```sql
SELECT pid, locktype, relation::REGCLASS, virtualxid AS virtxid, transactionid AS xid, mode, granted, pg_blocking_pids(pid) AS wait_for FROM pg_locks;
  pid  |   locktype    | relation | virtxid |  xid   |       mode       | granted | wait_for
-------+---------------+----------+---------+--------+------------------+---------+----------
 59837 | relation      | locks    |         |        | RowExclusiveLock | t       | {59397}
 59837 | virtualxid    |          | 7/68    |        | ExclusiveLock    | t       | {59397}
 59697 | relation      | pg_locks |         |        | AccessShareLock  | t       | {}
 59697 | virtualxid    |          | 6/135   |        | ExclusiveLock    | t       | {}
 59397 | relation      | locks    |         |        | RowExclusiveLock | t       | {}
 59397 | virtualxid    |          | 5/9     |        | ExclusiveLock    | t       | {}
 59837 | transactionid |          |         | 356061 | ShareLock        | f       | {59397}
 59397 | transactionid |          |         | 356061 | ExclusiveLock    | t       | {}
 59837 | transactionid |          |         | 356062 | ExclusiveLock    | t       | {59397}
(9 rows)
```
# Взаимная блокировка
Воспроизведем взаимоблокировку, для этого последовательно выполним следующим команды в указанных сеансах psql:
```sql
-- Первый сеанс
BEGIN;
update locks set caption = 'Раз' where id = 1;
-- Второй сеанс
BEGIN;
update locks set caption = 'Два' where id = 2;
-- Третий сеанс
BEGIN;
update locks set caption = 'Три' where id = 3;
-- Первый сеанс
update locks set caption = 'Двадва' where id = 2;
-- Второй сеанс
update locks set caption = 'Тритри' where id = 3;
-- Третий сеанс
update locks set caption = 'Разраз' where id = 1;
```
После чего получаем ошибку, сообщающую о наличии deadlock и отмену нашей транзакции, после которой выполняется ROLLBACK, даже при отправке команды COMMIT: 
```
ERROR:  deadlock detected
DETAIL:  Process 59397 waits for ShareLock on transaction 356070; blocked by process 59392.
Process 59392 waits for ShareLock on transaction 356071; blocked by process 59394.
Process 59394 waits for ShareLock on transaction 356072; blocked by process 59397.
HINT:  See server log for query details.
CONTEXT:  while updating tuple (0,16) in relation "locks"
```
В самом журнале PostgreSQL ошибка немного более развернутая и дает информацию о том какие запросы вызвали взаимоблокировку:
```
2022-10-27 05:25:27.934 UTC [59397] kyern@kyern ERROR:  deadlock detected
2022-10-27 05:25:27.934 UTC [59397] kyern@kyern DETAIL:  Process 59397 waits for ShareLock on transaction 356070; blocked by process 59392.
        Process 59392 waits for ShareLock on transaction 356071; blocked by process 59394.
        Process 59394 waits for ShareLock on transaction 356072; blocked by process 59397.
        Process 59397: update locks set caption = 'Разраз' where id = 1;
        Process 59392: update locks set caption = 'Двадва' where id = 2;
        Process 59394: update locks set caption = 'Тритри' where id = 3;
2022-10-27 05:25:27.934 UTC [59397] kyern@kyern HINT:  See server log for query details.
2022-10-27 05:25:27.934 UTC [59397] kyern@kyern CONTEXT:  while updating tuple (0,16) in relation "locks"
2022-10-27 05:25:27.934 UTC [59397] kyern@kyern STATEMENT:  update locks set caption = 'Разраз' where id = 1;
2022-10-27 05:25:27.935 UTC [59394] kyern@kyern LOG:  process 59394 acquired ShareLock on transaction 356072 after 4752.946 ms
2022-10-27 05:25:27.935 UTC [59394] kyern@kyern CONTEXT:  while updating tuple (0,18) in relation "locks"
2022-10-27 05:25:27.935 UTC [59394] kyern@kyern STATEMENT:  update locks set caption = 'Тритри' where id = 3;
```

# Взаимоблокировка UPDATE
Запустим два транзакции с обновлением всей таблицы без условия:
```sql
-- Сеанс 1
update locks set caption = 'All as one!';
-- Сеанс 2
update locks set caption = 'Not all as one!';
```
В журнале видим сообщение о длительной блокировке. Смотрим что по самим блокировкам:
```sql
SELECT pid, locktype, relation::REGCLASS, virtualxid AS virtxid, transactionid AS xid, mode, granted, pg_blocking_pids(pid) AS wait_for FROM pg_locks;
  pid  |   locktype    | relation | virtxid |  xid   |       mode       | granted | wait_for
-------+---------------+----------+---------+--------+------------------+---------+----------
 59697 | relation      | pg_locks |         |        | AccessShareLock  | t       | {}
 59697 | virtualxid    |          | 6/137   |        | ExclusiveLock    | t       | {}
 59394 | relation      | locks    |         |        | RowExclusiveLock | t       | {59392}
 59394 | virtualxid    |          | 4/7     |        | ExclusiveLock    | t       | {59392}
 59392 | relation      | locks    |         |        | RowExclusiveLock | t       | {}
 59392 | virtualxid    |          | 3/19041 |        | ExclusiveLock    | t       | {}
 59392 | transactionid |          |         | 356073 | ExclusiveLock    | t       | {}
 59394 | transactionid |          |         | 356074 | ExclusiveLock    | t       | {59392}
 59394 | transactionid |          |         | 356073 | ShareLock        | f       | {59392}
 59394 | tuple         | locks    |         |        | ExclusiveLock    | t       | {59392}
(10 rows)
```
Оба запроса получают эксклюзивную блокировку на один и тот же объект, два запроса не смогут создать блокировку в одно и то же время, всегда кто-то получит ее быстрее, чем второй.