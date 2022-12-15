# Домашнее задание к занятию "`12.5 Index`" - `Кузнецов Андрей`
### Задание 1
Напишите запрос к учебной базе данных, который вернет процентное отношение общего размера всех индексов к общему размеру всех таблиц.

```
select (sum(index_length) / sum(data_length) * 100) as Total_percent  from information_schema.tables where table_schema = 'sakila';
+---------------+
| Total_percent |
+---------------+
|       54.6816 |
+---------------+
1 row in set (0.00 sec)
```
---

### Задание 2
Выполните explain analyze следующего запроса:
```
select distinct concat(c.last_name, ' ', c.first_name), sum(p.amount) over (partition by c.customer_id, f.title)
from payment p, rental r, customer c, inventory i, film f
where date(p.payment_date) = '2005-07-30' and p.payment_date = r.rental_date and r.customer_id = c.customer_id and i.inventory_id = r.inventory_id
```

1-ый запуск неоптимизированного запроса:
```
| -> Table scan on <temporary>  (cost=2.50..2.50 rows=0) (actual time=6508.903..6508.950 rows=391 loops=1)
    -> Temporary table with deduplication  (cost=0.00..0.00 rows=0) (actual time=6508.900..6508.900 rows=391 loops=1)
        -> Window aggregate with buffering: sum(payment.amount) OVER (PARTITION BY c.customer_id,f.title )   (actual time=2955.187..6313.388 rows=642000 loops=1)
            -> Sort: c.customer_id, f.title  (actual time=2955.118..3016.722 rows=642000 loops=1)
                -> Stream results  (cost=21648276.28 rows=15587935) (actual time=205.699..2301.933 rows=642000 loops=1)
                    -> Nested loop inner join  (cost=21648276.28 rows=15587935) (actual time=205.680..2011.047 rows=642000 loops=1)
                        -> Nested loop inner join  (cost=20073894.84 rows=15587935) (actual time=177.464..1772.566 rows=642000 loops=1)
                            -> Nested loop inner join  (cost=18499513.41 rows=15587935) (actual time=61.692..1449.967 rows=642000 loops=1)
                                -> Inner hash join (no condition)  (cost=1540399.19 rows=15400000) (actual time=42.821..239.270 rows=634000 loops=1)
                                    -> Filter: (cast(p.payment_date as date) = '2005-07-30')  (cost=1.83 rows=15400) (actual time=27.014..185.451 rows=634 loops=1)
                                        -> Table scan on p  (cost=1.83 rows=15400) (actual time=26.989..183.738 rows=16044 loops=1)
                                    -> Hash
                                        -> Covering index scan on f using idx_title  (cost=112.00 rows=1000) (actual time=13.359..15.698 rows=1000 loops=1)
                                -> Covering index lookup on r using rental_date (rental_date=p.payment_date)  (cost=1.00 rows=1) (actual time=0.001..0.002 rows=1 loops=634000)
                            -> Single-row index lookup on c using PRIMARY (customer_id=r.customer_id)  (cost=0.00 rows=1) (actual time=0.000..0.000 rows=1 loops=642000)
                        -> Single-row covering index lookup on i using PRIMARY (inventory_id=r.inventory_id)  (cost=0.00 rows=1) (actual time=0.000..0.000 rows=1 loops=642000)
 |
```

Дальше я думаю можно убрать 2 узких горлышка, 1-ое это поиск уникальных значений, distinct, второе - это ненужная оконная функция over, ну и сгрупировать как-нибудь, например по искомому полю name. И уже видим что запрос обработался гораздо быстрее
```
explain analyze select concat(c.last_name, ' ', c.first_name) as name, sum(p.amount) from payment p, rental r, customer c, inventory i, film f where date(p.payment_date) = '2005;


| -> Table scan on <temporary>  (actual time=3462.727..3462.811 rows=391 loops=1)
    -> Aggregate using temporary table  (actual time=3462.723..3462.723 rows=391 loops=1)
        -> Nested loop inner join  (cost=21143371.42 rows=15587935) (actual time=0.265..1595.358 rows=642000 loops=1)
            -> Nested loop inner join  (cost=19580680.93 rows=15587935) (actual time=0.261..1400.607 rows=642000 loops=1)
                -> Nested loop inner join  (cost=18017990.44 rows=15587935) (actual time=0.256..1207.259 rows=642000 loops=1)
                    -> Inner hash join (no condition)  (cost=1540136.25 rows=15400000) (actual time=0.244..37.949 rows=634000 loops=1)
                        -> Filter: (cast(p.payment_date as date) = '2005-07-30')  (cost=1.56 rows=15400) (actual time=0.031..5.635 rows=634 loops=1)
                            -> Table scan on p  (cost=1.56 rows=15400) (actual time=0.021..4.208 rows=16044 loops=1)
                        -> Hash
                            -> Covering index scan on f using idx_fk_language_id  (cost=112.00 rows=1000) (actual time=0.027..0.152 rows=1000 loops=1)
                    -> Covering index lookup on r using rental_date (rental_date=p.payment_date)  (cost=0.97 rows=1) (actual time=0.001..0.002 rows=1 loops=634000)
                -> Single-row index lookup on c using PRIMARY (customer_id=r.customer_id)  (cost=0.00 rows=1) (actual time=0.000..0.000 rows=1 loops=642000)
            -> Single-row covering index lookup on i using PRIMARY (inventory_id=r.inventory_id)  (cost=0.00 rows=1) (actual time=0.000..0.000 rows=1 loops=642000)
 |

```
Следующая иттерация, отказываемся от ненужной таблицы film
```
explain analyze select  concat(c.last_name, ' ', c.first_name) as name, sum(p.amount) from payment p, rental r, customer c, inventory i where date(p.payment_date) = '2005-07-30' and p.payment_date = r.rental_date and  r.customer_id = c.customer_id and i.inventory_id = r.inventory_id group by name;

| -> Table scan on <temporary>  (actual time=7.922..7.963 rows=391 loops=1)
    -> Aggregate using temporary table  (actual time=7.920..7.920 rows=391 loops=1)
        -> Nested loop inner join  (cost=28953.66 rows=15588) (actual time=0.075..7.352 rows=642 loops=1)
            -> Nested loop inner join  (cost=23497.88 rows=15588) (actual time=0.072..6.664 rows=642 loops=1)
                -> Nested loop inner join  (cost=18042.10 rows=15588) (actual time=0.066..6.119 rows=642 loops=1)
                    -> Filter: (cast(p.payment_date as date) = '2005-07-30')  (cost=1564.25 rows=15400) (actual time=0.051..4.885 rows=634 loops=1)
                        -> Table scan on p  (cost=1564.25 rows=15400) (actual time=0.040..3.666 rows=16044 loops=1)
                    -> Covering index lookup on r using rental_date (rental_date=p.payment_date)  (cost=0.97 rows=1) (actual time=0.001..0.002 rows=1 loops=634)
                -> Single-row index lookup on c using PRIMARY (customer_id=r.customer_id)  (cost=0.25 rows=1) (actual time=0.001..0.001 rows=1 loops=642)
            -> Single-row covering index lookup on i using PRIMARY (inventory_id=r.inventory_id)  (cost=0.25 rows=1) (actual time=0.001..0.001 rows=1 loops=642)

```


---

