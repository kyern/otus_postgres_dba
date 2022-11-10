# Триггеры
## Триггерная функция
```pgsql
create or replace function pract_functions.tf_update_sales_stats()
returns trigger as 
$$
declare c_good_name   varchar(63);
begin 
	
	-- Добавляем новую продажу
	if TG_OP = 'INSERT' then
		select g.good_name into c_good_name from goods g where g.goods_id = NEW.good_id;
		if not exists (select gsm.good_name from good_sum_mart gsm where gsm.good_name = c_good_name) then  
			insert into good_sum_mart (good_name, sum_sale) values (c_good_name, 0);
		end if;
	
		update good_sum_mart gsm
			set sum_sale = sum_sale + (
				select NEW.sales_qty*g.good_price 
				from goods g where NEW.good_id = g.goods_id
			) 
		where gsm.good_name = c_good_name;
	-- Актуализируем статистику при изменении
	elsif TG_OP = 'UPDATE' then
		select g.good_name into c_good_name from goods g where g.goods_id = NEW.good_id;
	
		update good_sum_mart gsm
			set sum_sale = sum_sale - (
				select OLD.sales_qty*g.good_price 
				from goods g where OLD.good_id = g.goods_id
			) 
		where gsm.good_name = c_good_name;
	
		update good_sum_mart gsm
			set sum_sale = sum_sale + (
				select NEW.sales_qty*g.good_price 
				from goods g where NEW.good_id = g.goods_id
			) 
		where gsm.good_name = c_good_name;
	-- Актуализируем статистику при удалении
	elsif TG_OP = 'DELETE' then
		select g.good_name into c_good_name from goods g where g.goods_id = OLD.good_id;
	
		update good_sum_mart gsm
			set sum_sale = sum_sale - (
				select OLD.sales_qty*g.good_price 
				from goods g where OLD.good_id = g.goods_id
			) 
		where gsm.good_name = c_good_name;
	
	end if;
	-- Удаляем позиции по которым нет продаж
	delete from good_sum_mart where sum_sale = 0;
	return null;
end;
$$
language plpgsql


```
## Триггер
```pgsql
create trigger t_after_sale
after INSERT or UPDATE or DELETE
on pract_functions.sales 
for each row execute function pract_functions.tf_update_sales_stats();
```
## Заполнение
Так как триггер на сработает на уже внесенные записи в таблице sales, нам нужно самостоятельно заполнить good_sum_mart текущими данными:
```pgsql
insert into good_sum_mart
SELECT G.good_name, sum(G.good_price * S.sales_qty)
FROM goods G
INNER JOIN sales S ON S.good_id = G.goods_id
GROUP BY G.good_name;
```
## Зачем?
Для более быстрого получение отчетов по продажам. Таблица всегда находится в актуальном состоянии, не нужны кучи джоинов. Статистика учитывает произведенные изменения цем, а не пересчитывает продажи по текущим ценам.  
Но это же является и минусом. Сейчас не хранится нигде информация о цене по которой был куплен товар, что влияет на актуальную точносить при автуализации таблицы после апдейта sales. Например если вчера спички купили 10 коробков спичек по 10 рублей, сегодня они стоили стоить 100 рублей за единицу товара, то при уменьшении количества проданных товаров после изменения цены, мы по факту вычтем текущую цену товара, а не ту, которая была на момент его продажи.