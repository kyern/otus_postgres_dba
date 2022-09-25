# Контрольные точки
Установим в файле /etc/postgresql/14/bench/postgresql.conf значение параметра checkpoint_timeout равное 30s.  
Включим логгирование и перезапустим кластер: 
```pgsql
ALTER SYSTEM SET log_checkpoints = on;
```
Cбросим статистку чекпоинтов:
```pgsql
select pg_stat_reset_shared('bgwriter');
```
Узнаем текущую точку журнала:
```pgsql
postgres=# SELECT pg_current_wal_insert_lsn();
 pg_current_wal_insert_lsn
---------------------------
 0/AD373EC0
(1 row)
```
Подготовим и запустим тестирование pgbench:
```bash
pgbench -i -p 5433 postgres && pgbench -c8 -P 60 -T 600 -p 5433 postgres
```
За время выполнения тестирования выполнилcя 21 чекпоинт:
```pgsql
postgres=# SELECT * FROM pg_stat_bgwriter \gx
-[ RECORD 1 ]---------+-----------------------------
checkpoints_timed     | 21
checkpoints_req       | 0
checkpoint_write_time | 538871
checkpoint_sync_time  | 349
buffers_checkpoint    | 40727
buffers_clean         | 0
maxwritten_clean      | 0
buffers_backend       | 4710
buffers_backend_fsync | 0
buffers_alloc         | 4946
stats_reset           | 2022-09-25 11:56:28.21391+00
```
И информация из журнала подтверждает количество выполеннных контрольных точек и факт их выполнения точно по расписанию:
```log
2022-09-25 11:57:03.131 UTC [1510345] LOG:  checkpoint starting: time
2022-09-25 11:57:33.092 UTC [1510345] LOG:  checkpoint starting: time
2022-09-25 11:58:03.140 UTC [1510345] LOG:  checkpoint starting: time
2022-09-25 11:58:33.080 UTC [1510345] LOG:  checkpoint starting: time
2022-09-25 11:59:03.128 UTC [1510345] LOG:  checkpoint starting: time
2022-09-25 11:59:33.050 UTC [1510345] LOG:  checkpoint starting: time
2022-09-25 12:00:03.120 UTC [1510345] LOG:  checkpoint starting: time
2022-09-25 12:00:33.076 UTC [1510345] LOG:  checkpoint starting: time
2022-09-25 12:01:03.107 UTC [1510345] LOG:  checkpoint starting: time
2022-09-25 12:01:33.036 UTC [1510345] LOG:  checkpoint starting: time
2022-09-25 12:02:03.124 UTC [1510345] LOG:  checkpoint starting: time
2022-09-25 12:02:33.064 UTC [1510345] LOG:  checkpoint starting: time
2022-09-25 12:03:03.113 UTC [1510345] LOG:  checkpoint starting: time
2022-09-25 12:03:33.048 UTC [1510345] LOG:  checkpoint starting: time
2022-09-25 12:04:03.083 UTC [1510345] LOG:  checkpoint starting: time
2022-09-25 12:04:33.120 UTC [1510345] LOG:  checkpoint starting: time
2022-09-25 12:05:03.048 UTC [1510345] LOG:  checkpoint starting: time
2022-09-25 12:05:33.083 UTC [1510345] LOG:  checkpoint starting: time
2022-09-25 12:06:03.039 UTC [1510345] LOG:  checkpoint starting: time
2022-09-25 12:06:33.060 UTC [1510345] LOG:  checkpoint starting: time
2022-09-25 12:07:33.112 UTC [1510345] LOG:  checkpoint starting: time
```
Получим значение новой точки: 
```pgsql
postgres=# SELECT pg_current_wal_insert_lsn();
 pg_current_wal_insert_lsn
---------------------------
 0/CC5D9EB8
(1 row)
```
И рассчитаем объем журналов сгенерированный между ними:
```
SELECT pg_size_pretty('0/CC5D9EB8'::pg_lsn - '0/AD373EC0'::pg_lsn);
 pg_size_pretty
----------------
 498 MB
(1 row)
```
Средний размер одной контрольной точки составил: 498 МБ / 21 = 23.7 МБ

# Асинхронный режим коммита
При прошлом тестировании среднее значение количества транзакций в секунду составило 742.074832. Выключим синхронный коммит и повторим тестирование.
```pgsql
ALTER SYSTEM SET synchronous_commit TO off;
```
После его отключения, среднее значение TPS составило 2593.012693, то есть скорость выполнения транзакций выросла более чем в три раза за счет отсутсвия ожидания сохранения данных на диск, но это значительно сказывается на надежности системы, так как если сохранение на диск не завершится, PostgreSQL об этом не узнает и мы можем потерять данные. 
# Контрольная сумма страниц
Создадим новый кластер с включенной контрольной суммой страниц:
```bash
sudo pg_createcluster 14 checksum -- --data-checksums
sudo systemctl start postgresql@14-checksum
```
Создадим в нем новую таблицу и заполним ее некоторыми данными и посмотрим ее расположение на диске:
```pgsql
 create table asdf (c int);
 insert into asdf values (1),(2),(3),(4),(5);
 SELECT pg_relation_filepath('asdf');
```
Остановим кластер и изменим несколько байт в файлах таблицы:
```bash
sudo systemctl stop postgresql@14-checksum
sudo dd if=/dev/zero of=/var/lib/postgresql/14/checksum/base/13726/16384 oflag=dsync conv=notrunc bs=1024 count=8
```
Запустим кластер и проверим данные в таблице:
```bash
sudo systemctl start postgresql@14-checksum
```
```pgsql
SELECT * FROM asdf;
```
Запрос выполнился без ошибок. Скорее всего, dd попал в область значения которой уже забито только 0, подобрать нужное значение с помощью параметра seek мне не удалось. 
При включенной проверке контрольных сумм, нарушение их целостности генерирует ошибку, игнорировать которую можно включив параметр ignore_checksum_failure.