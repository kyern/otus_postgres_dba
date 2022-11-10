# Секционирование
## Анализ
Рассмотрим возможность секционирования таблицы `boarding_passes`, которая содержит информацию о посадочных талонах пассажиров.  
Проанализируем информацию о таблице:
```
demo=# \d boarding_passes
                  Table "bookings.boarding_passes"
   Column    |         Type         | Collation | Nullable | Default
-------------+----------------------+-----------+----------+---------
 ticket_no   | character(13)        |           | not null |
 flight_id   | integer              |           | not null |
 boarding_no | integer              |           | not null |
 seat_no     | character varying(4) |           | not null |
```
Наиболее частый способ использования таблицы посадочного талона - запись информации о новом посадочном талоне и печать посадочного талона пассажиром после прохождения регистрации на рейс. Наиболее логичным тут кажется партицирование на основе номера рейса. Так как к более новым посадочным талонам обращения будут значительно чаще (нет необходимости печатать посадочный талон на рейс, которые уже убыл), следовательно больше подойдет партицирование на основе интервалов. Наиболее идеальным было бы партицирование на основе номера самого посадочного талона, в крайнем случае номера билета. Но у самого посадочного талона нет номера, а номер билета хранится в формате строки.  
Всего у нас в таблице почти 2 миллиона записей. flight_id при этом находятся в интервале от 1 до 65662. Будем создавать одну секцию на каждые 5000 рейсов.
## Создание    
Создадим таблицу с секциами, идентичную по схеме оригинальной `boarding_passes`
```pgsql
CREATE TABLE bookings.boarding_passes_p (
	ticket_no bpchar(13) NOT NULL,
	flight_id int4 NOT NULL,
	boarding_no int4 NOT NULL,
	seat_no varchar(4) NOT NULL,
	CONSTRAINT boarding_passes_p_flight_id_boarding_no_key UNIQUE (flight_id, boarding_no),
	CONSTRAINT boarding_passes_p_flight_id_seat_no_key UNIQUE (flight_id, seat_no),
	CONSTRAINT boarding_passes_p_pkey PRIMARY KEY (ticket_no, flight_id),
	CONSTRAINT boarding_passes_p_ticket_no_fkey FOREIGN KEY (ticket_no,flight_id) REFERENCES bookings.ticket_flights(ticket_no,flight_id)
) PARTITION BY RANGE (flight_id);
```
Далее через psql создадим сразу все секции:
```pgsql
demo=# select 'CREATE TABLE bookings.boarding_passes_p_' || i || ' PARTITION OF bookings.boarding_passes_p FOR VALUES FROM ('|| ps ||') TO ('|| pe ||');'
demo-# from (SELECT g as i, g*5000 as ps, (g+1)*5000 as pe FROM generate_series(0, (select max(bp.flight_id) from boarding_passes bp)/5000) g) sec
demo-# \gexec
```
В результате будет выполнен следующий набор команд:  
```pgsql
CREATE TABLE bookings.boarding_passes_p_0 PARTITION OF bookings.boarding_passes_p FOR VALUES FROM (0) TO (5000);
CREATE TABLE bookings.boarding_passes_p_1 PARTITION OF bookings.boarding_passes_p FOR VALUES FROM (5000) TO (10000);
CREATE TABLE bookings.boarding_passes_p_2 PARTITION OF bookings.boarding_passes_p FOR VALUES FROM (10000) TO (15000);
CREATE TABLE bookings.boarding_passes_p_3 PARTITION OF bookings.boarding_passes_p FOR VALUES FROM (15000) TO (20000);
CREATE TABLE bookings.boarding_passes_p_4 PARTITION OF bookings.boarding_passes_p FOR VALUES FROM (20000) TO (25000);
CREATE TABLE bookings.boarding_passes_p_5 PARTITION OF bookings.boarding_passes_p FOR VALUES FROM (25000) TO (30000);
CREATE TABLE bookings.boarding_passes_p_6 PARTITION OF bookings.boarding_passes_p FOR VALUES FROM (30000) TO (35000);
CREATE TABLE bookings.boarding_passes_p_7 PARTITION OF bookings.boarding_passes_p FOR VALUES FROM (35000) TO (40000);
CREATE TABLE bookings.boarding_passes_p_8 PARTITION OF bookings.boarding_passes_p FOR VALUES FROM (40000) TO (45000);
CREATE TABLE bookings.boarding_passes_p_9 PARTITION OF bookings.boarding_passes_p FOR VALUES FROM (45000) TO (50000);
CREATE TABLE bookings.boarding_passes_p_10 PARTITION OF bookings.boarding_passes_p FOR VALUES FROM (50000) TO (55000);
CREATE TABLE bookings.boarding_passes_p_11 PARTITION OF bookings.boarding_passes_p FOR VALUES FROM (55000) TO (60000);
CREATE TABLE bookings.boarding_passes_p_12 PARTITION OF bookings.boarding_passes_p FOR VALUES FROM (60000) TO (65000);
CREATE TABLE bookings.boarding_passes_p_13 PARTITION OF bookings.boarding_passes_p FOR VALUES FROM (65000) TO (70000);
```
Дополнительно добавим секцию по умолчанию, чтобы забытая секция нам не привела к потере данных:
```pgsql
CREATE TABLE bookings.boarding_passes_p_d PARTITION OF bookings.boarding_passes_p DEFAULT;
```
И заполним нашу таблицу данными из уже существующей:
```pgsql
insert into boarding_passes_p select * from bookings.boarding_passes;
```
## Результаты
При запросе посадочных талонов для одного конкретного рейса секционированная таблица проигрывает в скорости оригинальной, так как идет дополнительный шаг поиска нужно секции и просто поиск в индексе происходит быстрее. В данном случае, теория не подтвердилась и ожидаемого выигрыша в производительности мы не получили.
```pgsql
-- Запрос по оригинальной таблице
explain analyze 
select * from boarding_passes bpp
where flight_id > 11000 and flight_id < 22000

Index Scan using boarding_passes_flight_id_seat_no_key on boarding_passes bpp  (cost=0.43..24717.46 rows=408298 width=25) (actual time=0.019..85.469 rows=411363 loops=1)
  Index Cond: ((flight_id > 11000) AND (flight_id < 22000))
Planning Time: 0.081 ms
Execution Time: 101.005 ms


-- Запрос по секционированной таблице
explain analyze
select * from boarding_passes_p bppd
where flight_id > 11000 and flight_id < 22000

Append  (cost=0.42..14932.12 rows=412336 width=25) (actual time=0.019..100.525 rows=411363 loops=1)
  ->  Index Scan using boarding_passes_p_2_flight_id_boarding_no_key on boarding_passes_p_2 bppd_1  (cost=0.42..5199.46 rows=132301 width=25) (actual time=0.019..25.815 rows=132366 loops=1)
        Index Cond: ((flight_id > 11000) AND (flight_id < 22000))
  ->  Seq Scan on boarding_passes_p_3 bppd_2  (cost=0.00..4743.49 rows=212166 width=25) (actual time=0.014..26.948 rows=212166 loops=1)
        Filter: ((flight_id > 11000) AND (flight_id < 22000))
  ->  Index Scan using boarding_passes_p_4_flight_id_seat_no_key on boarding_passes_p_4 bppd_3  (cost=0.42..2927.49 rows=67869 width=25) (actual time=0.023..15.298 rows=66831 loops=1)
        Index Cond: ((flight_id > 11000) AND (flight_id < 22000))
Planning Time: 0.254 ms
Execution Time: 116.014 ms
```

Но данный подход теоретически может дать выгоду при очистке старых посадочных талонов из базы.
```pgsql
-- Запрос по оригинальной таблице
explain analyze 
select * from boarding_passes bpp
where flight_id > 0 and flight_id < 10000

Index Scan using boarding_passes_flight_id_seat_no_key on boarding_passes bpp  (cost=0.43..24289.03 rows=389684 width=25) (actual time=0.016..88.777 rows=389700 loops=1)
  Index Cond: ((flight_id > 0) AND (flight_id < 10000))
Planning Time: 0.108 ms
Execution Time: 104.068 ms


-- Запрос по секционированной таблице
explain analyze
select * from boarding_passes_p bppd
where flight_id > 0 and flight_id < 10000

Append  (cost=0.00..10660.00 rows=389700 width=25) (actual time=0.007..81.109 rows=389700 loops=1)
  ->  Seq Scan on boarding_passes_p_0 bppd_1  (cost=0.00..5282.82 rows=236321 width=25) (actual time=0.006..30.257 rows=236321 loops=1)
        Filter: ((flight_id > 0) AND (flight_id < 10000))
  ->  Seq Scan on boarding_passes_p_1 bppd_2  (cost=0.00..3428.68 rows=153379 width=25) (actual time=0.011..19.411 rows=153379 loops=1)
        Filter: ((flight_id > 0) AND (flight_id < 10000))
Planning Time: 0.222 ms
Execution Time: 95.340 ms
```
