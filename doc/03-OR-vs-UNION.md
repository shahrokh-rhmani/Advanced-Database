### Scenario 1: Users from Tehran or Iran (with OR)
```
EXPLAIN ANALYZE
SELECT user_id, username, city, country
FROM users
WHERE city = 'Tehran' OR country = 'Iran';
```

### output:
```
 Seq Scan on users  (cost=0.00..28956.00 rows=238046 width=26) (actual time=0.020..130.445 rows=233960 loops=1)
   Filter: (((city)::text = 'Tehran'::text) OR ((country)::text = 'Iran'::text))
   Rows Removed by Filter: 766040
 Planning Time: 0.132 ms
 Execution Time: 137.597 ms
(5 rows)
```
```
\! clear
```

### Scenario 2: Same query with UNION
```
EXPLAIN ANALYZE
SELECT user_id, username, city, country
FROM users
WHERE city = 'Tehran'
UNION
SELECT user_id, username, city, country
FROM users
WHERE country = 'Iran';
```

### output:
```
 Unique  (cost=150896.80..154074.30 rows=254200 width=358) (actual time=204.832..287.822 rows=233960 loops=1)
   ->  Sort  (cost=150896.80..151532.30 rows=254200 width=358) (actual time=204.830..238.866 rows=249431 loops=1)
         Sort Key: users.user_id, users.username, users.city, users.country
         Sort Method: external merge  Disk: 9288kB
         ->  Append  (cost=0.00..44663.22 rows=254200 width=358) (actual time=12.670..164.653 rows=249431 loops=1)
               ->  Seq Scan on users  (cost=0.00..26456.00 rows=127000 width=26) (actual time=12.669..101.852 rows=124406 loops=1)
                     Filter: ((city)::text = 'Tehran'::text)
                     Rows Removed by Filter: 875594
               ->  Bitmap Heap Scan on users users_1  (cost=1390.22..16936.22 rows=127200 width=26) (actual time=6.054..48.439 rows=125025 loops=1)
                     Recheck Cond: ((country)::text = 'Iran'::text)
                     Heap Blocks: exact=13956
                     ->  Bitmap Index Scan on idx_users_country  (cost=0.00..1358.42 rows=127200 width=0) (actual time=4.257..4.257 rows=125025 loops=1)
                           Index Cond: ((country)::text = 'Iran'::text)
 Planning Time: 0.112 ms
 JIT:
   Functions: 9
   Options: Inlining false, Optimization false, Expressions true, Deforming true
   Timing: Generation 0.606 ms, Inlining 0.000 ms, Optimization 1.457 ms, Emission 11.050 ms, Total 13.114 ms
 Execution Time: 299.820 ms
(19 rows)
```

