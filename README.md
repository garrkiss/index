# Домашнее задание к занятию "`Индексы`" - `Бакулев Евгений`

### Задание 1
### Что нужно сделать:

1. Напишите запрос к учебной базе данных, который вернёт процентное отношение общего размера всех индексов к общему размеру всех таблиц.

### Решение 1

![Скрин](https://github.com/garrkiss/index/blob/main/img/%D0%97%D0%B0%D0%B4%D0%B0%D0%BD%D0%B8%D0%B51.png)

### Задание 2
### Что нужно сделать:

1. Выполните explain analyze следующего запроса:
   
```
select distinct concat(c.last_name, ' ', c.first_name), sum(p.amount) over (partition by c.customer_id, f.title)
from payment p, rental r, customer c, inventory i, film f
where date(p.payment_date) = '2005-07-30' and p.payment_date = r.rental_date and r.customer_id = c.customer_id and i.inventory_id = r.inventory_id
```
2. перечислите узкие места;

3.  оптимизируйте запрос: внесите корректировки по использованию операторов, при необходимости добавьте индексы.


### Решение 2

**Выполненный explain analyze исходного запроса**

```
-> Table scan on <temporary>  (cost=2.50..2.50 rows=0) (actual time=3473.669..3473.713 rows=391 loops=1)
    -> Temporary table with deduplication  (cost=0.00..0.00 rows=0) (actual time=3473.666..3473.666 rows=391 loops=1)
        -> Window aggregate with buffering: sum(payment.amount) OVER (PARTITION BY c.customer_id,f.title )   (actual time=1404.025..3358.806 rows=642000 loops=1)
            -> Sort: c.customer_id, f.title  (actual time=1403.992..1452.532 rows=642000 loops=1)
                -> Stream results  (cost=21711641.73 rows=16009975) (actual time=0.338..1069.441 rows=642000 loops=1)
                    -> Nested loop inner join  (cost=21711641.73 rows=16009975) (actual time=0.332..934.265 rows=642000 loops=1)
                        -> Nested loop inner join  (cost=20106641.71 rows=16009975) (actual time=0.328..802.396 rows=642000 loops=1)
                            -> Nested loop inner join  (cost=18501641.70 rows=16009975) (actual time=0.322..670.094 rows=642000 loops=1)
                                -> Inner hash join (no condition)  (cost=1581474.80 rows=15813000) (actual time=0.310..32.644 rows=634000 loops=1)
                                    -> Filter: (cast(p.payment_date as date) = '2005-07-30')  (cost=1.65 rows=15813) (actual time=0.032..4.531 rows=634 loops=1)
                                        -> Table scan on p  (cost=1.65 rows=15813) (actual time=0.021..3.154 rows=16044 loops=1)
                                    -> Hash
                                        -> Covering index scan on f using idx_title  (cost=103.00 rows=1000) (actual time=0.037..0.205 rows=1000 loops=1)
                                -> Covering index lookup on r using rental_date (rental_date=p.payment_date)  (cost=0.97 rows=1) (actual time=0.001..0.001 rows=1 loops=634000)
                            -> Single-row index lookup on c using PRIMARY (customer_id=r.customer_id)  (cost=0.00 rows=1) (actual time=0.000..0.000 rows=1 loops=642000)
                        -> Single-row covering index lookup on i using PRIMARY (inventory_id=r.inventory_id)  (cost=0.00 rows=1) (actual time=0.000..0.000 rows=1 loops=642000)
```




**Изменённый запрос**

```
EXPLAIN ANALYZE
SELECT DISTINCT
    CONCAT(c.last_name, ' ', c.first_name) AS customer_name,
    SUM(p.amount) OVER (PARTITION BY c.customer_id, f.title) AS total_amount
FROM
    payment p
    JOIN rental r ON p.payment_date = r.rental_date
    JOIN customer c ON r.customer_id = c.customer_id
    JOIN inventory i ON i.inventory_id = r.inventory_id
    JOIN film f ON i.film_id = f.film_id
WHERE
    p.payment_date >= '2005-07-30 00:00:00'
    AND p.payment_date < '2005-07-31 00:00:00';
```

**Выполненный explain analyze изменённого запроса**
```
-> Table scan on <temporary>  (cost=2.50..2.50 rows=0) (actual time=9.523..9.610 rows=602 loops=1)
    -> Temporary table with deduplication  (cost=0.00..0.00 rows=0) (actual time=9.521..9.521 rows=602 loops=1)
        -> Window aggregate with buffering: sum(payment.amount) OVER (PARTITION BY c.customer_id,f.title )   (actual time=8.361..9.368 rows=642 loops=1)
            -> Sort: c.customer_id, f.title  (actual time=8.333..8.380 rows=642 loops=1)
                -> Stream results  (cost=5352.65 rows=1779) (actual time=0.077..8.135 rows=642 loops=1)
                    -> Nested loop inner join  (cost=5352.65 rows=1779) (actual time=0.071..7.875 rows=642 loops=1)
                        -> Nested loop inner join  (cost=4730.16 rows=1779) (actual time=0.066..7.143 rows=642 loops=1)
                            -> Nested loop inner join  (cost=4107.68 rows=1779) (actual time=0.063..6.449 rows=642 loops=1)
                                -> Nested loop inner join  (cost=3485.19 rows=1779) (actual time=0.055..5.927 rows=642 loops=1)
                                    -> Filter: ((p.payment_date >= TIMESTAMP'2005-07-30 00:00:00') and (p.payment_date < TIMESTAMP'2005-07-31 00:00:00'))  (cost=1605.55 rows=1757) (actual time=0.045..4.738 rows=634 loops=1)
                                        -> Table scan on p  (cost=1605.55 rows=15813) (actual time=0.032..3.602 rows=16044 loops=1)
                                    -> Covering index lookup on r using rental_date (rental_date=p.payment_date)  (cost=0.97 rows=1) (actual time=0.001..0.002 rows=1 loops=634)
                                -> Single-row index lookup on c using PRIMARY (customer_id=r.customer_id)  (cost=0.25 rows=1) (actual time=0.001..0.001 rows=1 loops=642)
                            -> Single-row index lookup on i using PRIMARY (inventory_id=r.inventory_id)  (cost=0.25 rows=1) (actual time=0.001..0.001 rows=1 loops=642)
                        -> Single-row index lookup on f using PRIMARY (film_id=i.film_id)  (cost=0.25 rows=1) (actual time=0.001..0.001 rows=1 loops=642)

```

**Узкие места**

1. Использование DATE(p.payment_date) вместо диапазона дат, не используется индекс
2. Не используется JOIN:
3. Нет индекса payment(payment_date);