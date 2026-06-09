# Advanced-Databases-Assignment-2-Sofiia-Churikova

## 1. Setup
I ran `load_assignment_generator.py` script, it created `student_perf_lab` db, built CRM / e-commerce tables, inserted test data, ran parallel workers. While the load was running I connected to the same db and analyzed the behaviour using `EXPLAIN (ANALYZE, BUFFERS)`, `pg_stat_activity`, `pg_locks`,
`pg_blocking_pids` and `pg_stat_user_tables`.

## 2. Task 1 — Identifying the problems
### 2.1 Table sizes (finding the "wide table")
```sql
SELECT relname AS table_name, n_live_tup AS rows, pg_size_pretty(pg_total_relation_size(relid)) AS total_size
FROM pg_stat_user_tables ORDER BY n_live_tup DESC;
```

| table_name | rows | total_size |
|---|---:|---:|
| order_items | 359,509 | 45 MB |
| customer_events_wide | 200,000 | 83 MB |
| orders | 120,000 | 17 MB |
| customers | 20,000 | 6,648 kB |
| products | 2,000 | 544 kB |

**Finding:** `customer_events_wide` has only 200,000 rows but is the second-biggest table on
disk (83 MB). it has 24 mostly-`TEXT` columns.
This is the wide-table problem and it became my main bottleneck.

<img width="1164" height="348" alt="image" src="https://github.com/user-attachments/assets/593bb631-9a41-4c8c-932c-56cd00e1e1a8" />

### 2.2 Which queries create the most load
I used two sources of evidence.

**(a) The load script's console output** prints the time of each query.
`cartesian_pressure` (the 3-table join) took **~200 seconds** under concurrent load,
while the others were in seconds/milliseconds

<img width="756" height="966" alt="image" src="https://github.com/user-attachments/assets/de526b9e-cb44-4494-9704-24e1148ce1e7" />


**(b) `pg_stat_activity`** shows the longest currently-running queries and which ones are
waiting on a lock

```sql
SELECT pid, state, now() - query_start AS running_for, wait_event_type, query
FROM pg_stat_activity
WHERE state = 'active' AND query NOT ILIKE '%pg_stat_activity%'
ORDER BY running_for DESC;
```

<img width="2518" height="358" alt="image" src="https://github.com/user-attachments/assets/84103dd8-941a-4531-81c3-8331d69a960a" />

<img width="2532" height="110" alt="image" src="https://github.com/user-attachments/assets/bd517d23-bc87-4012-9b33-093301200545" />

<img width="2526" height="120" alt="image" src="https://github.com/user-attachments/assets/1da63eda-67e4-43e9-9ac2-e10659a3fd06" />

<img width="2530" height="224" alt="image" src="https://github.com/user-attachments/assets/b3f4e95d-a063-4ba5-a2c1-ad6f533ada20" />

<img width="2544" height="116" alt="image" src="https://github.com/user-attachments/assets/460dccf2-10e0-4105-bc46-e0bced520486" />

The top rows were the `cartesian_pressure` join and several `UPDATE` statements, and some
sessions showed `wait_event_type = Lock`

**Conclusion of Task 1:** the heaviest load comes from (1) the `cartesian_pressure` 3-table
join over the wide table, (2) aggregations doing a full Sequential Scan over `customer_events_wide`,
and (3) updates on the wide table.


## 3. Baseline — execution plans of the 6 queries (BEFORE)
I recorded scan type, time and buffers for each query before changing anything. Indexes that
already existed at the start were the ones the script creates itself: `idx_orders_customer_id`,
`idx_order_items_order_id`, `idx_order_items_product_id`, `idx_events_customer_id`.

**email**
```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT * FROM customers WHERE email LIKE '%gmail%';
```
<img width="1738" height="502" alt="image" src="https://github.com/user-attachments/assets/7ac80c02-a8c4-4858-81a1-1e0bffefd623" />

**orders city + status**
```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT * FROM orders WHERE delivery_city LIKE '%a%' AND status = 'paid';
```
<img width="1740" height="504" alt="image" src="https://github.com/user-attachments/assets/7ac40006-cd75-4515-acfa-17a8144ab96c" />

**heavy join**
```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT c.customer_id, c.full_name, COUNT(o.order_id) AS orders_count, SUM(o.total_amount) AS revenue
FROM customers c JOIN orders o ON c.customer_id = o.customer_id
WHERE c.status = 'active'
GROUP BY c.customer_id, c.full_name ORDER BY revenue DESC LIMIT 100;
```
<img width="1754" height="1478" alt="image" src="https://github.com/user-attachments/assets/ae11793f-4287-4b16-b1f8-1284de89456e" />

**events aggregation**
```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT customer_id, event_type, COUNT(*) AS events_count, MAX(event_time) AS last_event_time
FROM customer_events_wide
WHERE event_time >= NOW() - INTERVAL '180 days'
GROUP BY customer_id, event_type ORDER BY events_count DESC LIMIT 200;
```
<img width="1746" height="1046" alt="image" src="https://github.com/user-attachments/assets/cfea72b8-ecec-487e-9573-5bbede90e8db" />

***items * products**
```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT p.category, COUNT(*) AS items_sold, SUM(oi.quantity * oi.unit_price) AS revenue
FROM order_items oi JOIN products p ON oi.product_id = p.product_id
GROUP BY p.category ORDER BY revenue DESC;
```
<img width="1756" height="1862" alt="image" src="https://github.com/user-attachments/assets/cccc096a-748f-4822-955d-4ff3bf1d1c97" />
<img width="1748" height="1882" alt="image" src="https://github.com/user-attachments/assets/a19471b1-fd3e-4efa-a714-27832f99c385" />

**triple join**
```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT COUNT(*)
FROM customers c
JOIN orders o ON o.customer_id = c.customer_id
JOIN customer_events_wide e ON e.customer_id = c.customer_id
WHERE c.status IN ('active','inactive') AND e.event_time >= NOW() - INTERVAL '90 days';
```
<img width="2846" height="1844" alt="image" src="https://github.com/user-attachments/assets/9f35a5ae-81cf-42f3-9bb6-fe32666508c7" />


| Query | Scan / plan | Execution time | Buffers | Problem |
|---|---|---:|---:|---|
| **Q1** email `LIKE '%gmail%'` | Seq Scan on customers | 5.910 ms | 657 | leading-wildcard `LIKE` can't use B-tree |
| **Q2** orders city + status | Seq Scan on orders, removed 103,631 | 18.973 ms | 1,223 | no index on `orders.status` |
| **Q3** heavy join | Hash Join + 2 Seq Scans | 34.093 ms | 1,883 | reads ~1/3 of both tables |
| **Q4** events aggregation | Seq Scan on wide table, removed 101,341 | 174.467 ms | 9,239 | wide table scanned for 3 columns |
| **Q5** items × products | Parallel Seq Scan on order_items | 69.141 ms | 2,537 | full aggregation, scan is unavoidable |
| **Q6** triple join | Parallel Seq Scan on wide table, removed 75,240 | 72.201 ms | 9,255 | wide table again |

The two queries that hit `customer_events_wide` (Q4, Q6) read ~9,200 buffers each - that is the
whole 83 MB table. This is the clearest problem

## 4. Task 3/4/5 — Optimizations (BEFORE → AFTER)
### 4.1 (Q2) Index on `orders.status`
```sql
CREATE INDEX idx_orders_status ON orders(status);
ANALYZE orders;
```
```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT * FROM orders WHERE delivery_city LIKE '%a%' AND status = 'paid';
```
<img width="2838" height="834" alt="image" src="https://github.com/user-attachments/assets/4c72ca2c-d893-458f-bffb-e94abc838e04" />

| | Before | After |
|---|---|---|
| Scan | Seq Scan on orders | Bitmap Index Scan on `idx_orders_status` |
| Time | 18.973 ms | **8.290 ms** |

**What I did:** added an index on the selective column (`status`). ~2.3× faster.

### 4.2 (Q1) Substring search with a GIN trigram index
A B-tree cannot serve `LIKE '%...%'` because the pattern starts with `%`. B-tree only helps `LIKE 'foo%'`. The correct tool is a GIN index with `pg_trgm`.

```sql
CREATE INDEX idx_customers_email_trgm ON customers USING gin (email gin_trgm_ops);
ANALYZE customers;
```

```sql
EXPLAIN (ANALYZE, BUFFERS) SELECT * FROM customers WHERE email LIKE '%gmail%';
```

<img width="2820" height="668" alt="image" src="https://github.com/user-attachments/assets/58207494-cbf2-403f-9497-fd9a7757d752" />

```sql
EXPLAIN (ANALYZE, BUFFERS) SELECT * FROM customers WHERE email LIKE '%user12345@%';
```

<img width="2408" height="676" alt="image" src="https://github.com/user-attachments/assets/f176f1c5-c295-4cb3-a91a-1c0566c93980" />


In my generated data very few customers have a `gmail` address, so the filter is **highly
selective** — exactly the case where an index helps. The plan switched from a full Seq Scan to
a `Bitmap Index Scan`.

| | Before | After |
|---|---|---|
| Scan | Seq Scan on customers | Bitmap Index Scan on `idx_customers_email_trgm` |
| Buffers | 657 | **7** |
| Time | 5.910 ms | **0.081 ms** |

**What I did:** used the GIN for substring search. Because the filter is selective, the planner uses it and avoids reading the whole table.

> an index helps only when the filter returns few rows (high selectivity)


### 4.3 (Q4/Q6) Attempt: index on `event_time`
```sql
CREATE INDEX idx_events_event_time ON customer_events_wide(event_time);
ANALYZE customer_events_wide;
```
```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT customer_id, event_type, COUNT(*), MAX(event_time)
FROM customer_events_wide
WHERE event_time >= NOW() - INTERVAL '180 days'
GROUP BY customer_id, event_type ORDER BY 3 DESC LIMIT 200;
```

<img width="2470" height="1054" alt="image" src="https://github.com/user-attachments/assets/1c53ef84-de16-415b-bb5b-4ffd0360034b" />

For Q4 the planner **ignored** this index and kept the Seq Scan, because the filter
`event_time >= NOW() - 180 days` matches ~half the rows (it removed only 101,343 of 200,000).
At ~50% selectivity a Seq Scan is cheaper than an index.

| Q4 | Before | After event_time index |
|---|---|---|
| Scan | Seq Scan | **Seq Scan (index not used)** |
| Time | 174.467 ms | 163.198 ms |

**What I learned:** adding an index on a non-selective filter does not help — PostgreSQL
correctly keeps the Seq Scan. The real fix for Q4 is the wide table itself (next step)

### 4.4 (Q4/Q6 — main fix) Normalize / split the wide table
The analytical queries only use 3 of the 24 columns, so I moved the "hot" columns into a narrow
table.

```sql
CREATE TABLE customer_events_core (
    event_id    INT PRIMARY KEY,
    customer_id INT,
    event_type  TEXT,
    event_time  TIMESTAMP
);
INSERT INTO customer_events_core (event_id, customer_id, event_type, event_time)
SELECT event_id, customer_id, event_type, event_time FROM customer_events_wide;

CREATE INDEX idx_core_event_time  ON customer_events_core(event_time);
CREATE INDEX idx_core_customer_id ON customer_events_core(customer_id);
ANALYZE customer_events_core;
```

```sql
SELECT 'wide' AS t, pg_size_pretty(pg_table_size('customer_events_wide'))
UNION ALL SELECT 'core', pg_size_pretty(pg_table_size('customer_events_core'));
```

<img width="764" height="178" alt="image" src="https://github.com/user-attachments/assets/8a7f270a-9e17-486f-8bd3-2e744756bb1a" />

```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT customer_id, event_type, COUNT(*), MAX(event_time)
FROM customer_events_core
WHERE event_time >= NOW() - INTERVAL '180 days'
GROUP BY customer_id, event_type ORDER BY 3 DESC LIMIT 200;
```

<img width="2368" height="1034" alt="image" src="https://github.com/user-attachments/assets/390dc16d-0f81-4508-8e35-8159fe883c72" />

Size comparison:
| table | size |
|---|---:|
| customer_events_wide | 73 MB |
| customer_events_core | **11 MB** |

Re-running Q4 against the narrow table:
 
| Q4 | Wide table | Narrow `customer_events_core` |
|---|---:|---:|
| Buffers | 9,239 | **1,287** |
| Time | 174.467 ms | 147.262 ms |

**What I did:** normalized the wide table (kept only the columns the queries need). Buffer reads
dropped ~7× (9,239 → 1,287).
