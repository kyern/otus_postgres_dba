1. Создайте новый кластер PostgresSQL 14 (на выбор - GCE, CloudSQL)
    > sudo pg_createcluster 14 schema && sudo pg_ctlcluster 14 schema start
2. Зайдите в созданный кластер под пользователем postgres
    > sudo -u postgres psql
3. Создайте новую базу данных testdb
    > CREATE DATABASE testdb;
4. Зайдите в созданную базу данных под пользователем postgres
    > \c testdb
5. Создайте новую схему testnm
    > create schema testnm;
6. Создайте новую таблицу t1 с одной колонкой c1 типа integer
    > create table t1 (c1 integer);
7. Вставьте строку со значением c1=1
    > insert into t1 values (1);
8. Создайте новую роль readonly
    > create user readonly with nologin;
9. Дайте новой роли право на подключение к базе данных testdb
    > grant CONNECT ON DATABASE testdb TO readonly ;
10. Дайте новой роли право на использование схемы testnm
    > grant USAGE ON SCHEMA testnm TO readonly ;
11. Дайте новой роли право на select для всех таблиц схемы testnm
    > grant SELECT ON ALL tables in schema testnm TO readonly ;
12. Создайте пользователя testread с паролем test123
    > CREATE USER testread with password 'test123';
13. Дайте роль readonly пользователю testread
    > grant readonly TO testread;
14. Зайдите под пользователем testread в базу данных testdb
    > psql -U testread -h localhost -d testdb
15. Сделайте select * from t1;
    > select * from t1;
16. Получилось? (могло если вы делали сами не по шпаргалке и не упустили один существенный момент про который позже)
    > ERROR:  permission denied for table t1
17. Напишите что именно произошло в тексте домашнего задания
    > Ошибка из-за нехватки доступа к объекту.
18. У вас есть идеи почему? ведь права то дали?
    > search_path привел нас в public, так как схемы идентичной имени пользователя не было.
19. Посмотрите на список таблиц
    ```
    \dt  
            List of relations
    Schema | Name | Type  |  Owner
    --------+------+-------+----------
    public | t1   | table | postgres
    (1 row)
    ```
20. Подсказка в шпаргалке под пунктом 20
    > schema = public
21. А почему так получилось с таблицей (если делали сами и без шпаргалки то может у вас все нормально)
    > search_path привел нас в public, так как схемы идентичной имени пользователя не было.
22. Вернитесь в базу данных testdb под пользователем postgres
    > sudo -u postgres psql
23. Удалите таблицу t1
    >  drop table t1;
24. Создайте ее заново но уже с явным указанием имени схемы testnm
    >  create table testnm.t1 (c1 integer);
25. Вставьте строку со значением c1=1
    > insert into testnm.t1 values (1);
26. Зайдите под пользователем testread в базу данных testdb
    > psql -U testread -h localhost -d testdb
27. Сделайте select * from testnm.t1;
    > select * from testnm.t1;
28. Получилось?
    > Нет
29. Есть идеи почему? если нет - смотрите шпаргалку
    > `grant SELECT ON ALL tables in schema` "разворачивает" список текущих таблиц и дает права на каждую, а таблица пересоздавалась.
30. Как сделать так чтобы такое больше не повторялось? если нет идей - смотрите шпаргалку
    > Изменить привилегии назначаемые при создании объектов. ALTER default privileges in SCHEMA testnm grant SELECT on TABLEs to readonly; 
31. Сделайте select * from testnm.t1;
    > select * from testnm.t1;
32. Получилось?
    > Нет
33. Есть идеи почему? если нет - смотрите шпаргалку
    > Изменение дефолтных разрешений не меняет сущетвующие объекты.
34. Сделайте select * from testnm.t1;
    > select * from testnm.t1;
35. Получилось?
    > Да
36. Ура!
    > Ура!
37. Теперь попробуйте выполнить команду create table t2(c1 integer); insert into t2 values (2);
    > create table t2(c1 integer); insert into t2 values (2);
38. А как так? нам же никто прав на создание таблиц и insert в них под ролью readonly?
    > Роль public назначается всем пользователям по умолчанию и им разрешается создание объектов в схеме public.
39. Есть идеи как убрать эти права? если нет - смотрите шпаргалку
    > Отозвать права  
    > `revoke CREATE on SCHEMA public FROM public;`  
    > `revoke all on DATABASE testdb FROM public;`
40. Если вы справились сами то расскажите что сделали и почему, если смотрели шпаргалку - объясните что сделали и почему выполнив указанные в ней команды
    > Мы создали схему `testnm` и дали прав на ее использование и чтение роли `readonly`, так же дали права на чтение для всех таблиц, которые будут созданы в дальнейшем в схеме `testnm`. При этом мы запретили создание объектов в схеме `public` роли `public`.
41. Теперь попробуйте выполнить команду create table t3(c1 integer); insert into t2 values (2);
    > create table t3(c1 integer); insert into t2 values (2);
42. Расскажите что получилось и почему 
    > Создание новой таблицы завершилось ошибкой, так как права на эту операцию мы забрали, но добавление новых записей в t2 выполнилось успешно, так как мы являемся владельцами данного объекта