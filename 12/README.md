# Начальные значения и производительность
Количество транзакций в секунду будет фиксироваться в двух режимах:  
1.  Без установки нового соединения для кажой транзакции (команда `pgbench -c 50 -j 2 -P 10 -T 30 -M extended`)  
2.  С установкой нового соединения для каждой транзакции (команда `pgbench -c 50 -C -j 2 -P 10 -T 30 -M extended`)  

На только что созданном кластере, без изменения каких-либо настроек, были полученые следующие значения:
1. Без установки нового соединения для каждой транзакции среднее значение TPS составило `660.81`.  
2. С установкой нового соединения для каждой транзакции среднее значение TPS составило `207.11`.  

# PgTune
Применим значение параметров которые нам предлагает PgTune для нашей системы. В моем случае его рекомендации выглядят следующим образом:
```
# DB Version: 14
# OS Type: linux
# DB Type: web
# Total Memory (RAM): 4 GB
# CPUs num: 2
# Connections num: 60
# Data Storage: ssd

max_connections = 60
shared_buffers = 1GB
effective_cache_size = 3GB
maintenance_work_mem = 256MB
checkpoint_completion_target = 0.9
wal_buffers = 16MB
default_statistics_target = 100
random_page_cost = 1.1
effective_io_concurrency = 200
work_mem = 17476kB
min_wal_size = 1GB
max_wal_size = 4GB
max_worker_processes = 2
max_parallel_workers_per_gather = 1
max_parallel_workers = 2
max_parallel_maintenance_workers = 1
```
Запишем их в файл `/etc/postgresql/14/main/conf.d/pgtune.conf` и перезапустим кластер.
Проверим что значения из конфига применились:
```
$ sudo -u postgres psql -c "select name, setting, source, sourcefile from pg_settings where sourcefile = '/etc/postgresql/14/main/conf.d/pgtune.conf';"
               name               | setting |       source       |                 sourcefile
----------------------------------+---------+--------------------+--------------------------------------------
 checkpoint_completion_target     | 0.9     | configuration file | /etc/postgresql/14/main/conf.d/pgtune.conf
 default_statistics_target        | 100     | configuration file | /etc/postgresql/14/main/conf.d/pgtune.conf
 effective_cache_size             | 393216  | configuration file | /etc/postgresql/14/main/conf.d/pgtune.conf
 effective_io_concurrency         | 200     | configuration file | /etc/postgresql/14/main/conf.d/pgtune.conf
 maintenance_work_mem             | 262144  | configuration file | /etc/postgresql/14/main/conf.d/pgtune.conf
 max_connections                  | 60      | configuration file | /etc/postgresql/14/main/conf.d/pgtune.conf
 max_parallel_maintenance_workers | 1       | configuration file | /etc/postgresql/14/main/conf.d/pgtune.conf
 max_parallel_workers             | 2       | configuration file | /etc/postgresql/14/main/conf.d/pgtune.conf
 max_parallel_workers_per_gather  | 1       | configuration file | /etc/postgresql/14/main/conf.d/pgtune.conf
 max_wal_size                     | 4096    | configuration file | /etc/postgresql/14/main/conf.d/pgtune.conf
 max_worker_processes             | 2       | configuration file | /etc/postgresql/14/main/conf.d/pgtune.conf
 min_wal_size                     | 1024    | configuration file | /etc/postgresql/14/main/conf.d/pgtune.conf
 random_page_cost                 | 1.1     | configuration file | /etc/postgresql/14/main/conf.d/pgtune.conf
 shared_buffers                   | 131072  | configuration file | /etc/postgresql/14/main/conf.d/pgtune.conf
 wal_buffers                      | 2048    | configuration file | /etc/postgresql/14/main/conf.d/pgtune.conf
 work_mem                         | 17476   | configuration file | /etc/postgresql/14/main/conf.d/pgtune.conf
(16 rows)
```
Все установленные нами параметры на месте. Проведем замеры производительности. Результаты следующие:
|  |Соединение на клиента|Соединения на транзакцию|
|--|--|--|
|По умолчанию|660.81|207.11|
|PGTune|707.00|210.78|

# Оптимизируем дальше
Откатим изменения от PgTune переименовав файл в `pgtune.conf.noapply` и начнем тестирование остальных параметров.
## fsync
|  |Соединение на клиента|Соединения на транзакцию|
|--|--|--|
|По умолчанию|660.81|207.11|
|fsync off|1853.08|263.97|
## synchronous_commit
|  |Соединение на клиента|Соединения на транзакцию|
|--|--|--|
|По умолчанию|660.81|207.11|
|synchronous_commit off |1883.54|263.22|
# Все вместе
Вернем параметры PgTune и применим опции `fsync = off` и `synchronous_commit off` одновременно. Получим следующие результаты:
|  |Соединение на клиента|Соединения на транзакцию|
|--|--|--|
|По умолчанию|660.81|207.11|
|Максимальное быстродействие |1884.32|248.96|

# Выводы
1. Отключения fsync и синхронного коммита позволяют увеличить производительность кластера в 3 раза, жертвуя при этом надежностью.
2. Создание новых соединений для каждой транзакции - безумно дорогая операция, сводящая на нет любые попытки оптимизации производительности.
3. 1900 транзакций в секунду - предел моего домашнего ПК :)
4. На синтетических тестах почти невозможно оценить влияние параметров установленных по рекомендации PgTune, в то время как `workmem` и `shared_buffers` могут оказаться одними из ключевых параметров на рабочей БД.