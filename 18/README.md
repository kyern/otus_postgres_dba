# Соединения таблиц
Все запросы написаны для демонастрационной базы данных Flights https://postgrespro.com/community/demodb
## Прямое соединение
Показывает название аэропорта, город в котором он находится и количество вылетевших рейсов, которые были совершены 13 июня 2017 года.
```pgsql
select city,
    airport_name,
    count(status)
from airports a,
    flights f
where a.airport_code = f.departure_airport
    and f.scheduled_departure::DATE = '2017-06-13'
group by city,
    airport_name;
```
## Левостороннее соединение двух таблиц
Показывает общее количество вылетов из каждого аэропорта за все время.
```pgsql
select airport_name,
    count(f.flight_id)
from airports a
    left join flights f on a.airport_code = f.departure_airport
group by 1
order by 2 desc
```
## Кросс соединение двух таблиц
Выводит все возможные направления перелетов
```pgsql
select a.airport_name,
    a2.airport_name
from airports a
    cross join airports a2
where a.airport_code <> a2.airport_code
```
## Полное соединение двух таблиц
Выводит количество мест в каждой модели самолета
```pgsql
select a.model,
    count(s.seat_no)
from seats s
    full join aircrafts a on a.aircraft_code = s.aircraft_code
group by 1
```
## Запрос, в котором будут использованы разные типы соединений
Выводит информацию по каждому месту масолета для конкретного рейса. 
```pgsql
select distinct s.seat_no,
    t.passenger_name
from flights f
    join seats s on s.aircraft_code = f.aircraft_code
    full join boarding_passes bp on bp.flight_id = f.flight_id
    and bp.seat_no = s.seat_no
    full join tickets t on t.ticket_no = bp.ticket_no
where f.flight_id = 19506
order by s.seat_no
```