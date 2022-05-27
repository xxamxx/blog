# 比较INSERT DO UPDATE和临时表批量更新



先准备千万级数据


```sql
INSERT INTO t (id, name) SELECT i, MD5(i::VARCHAR) FROM generate_series(1, 10000000) t(i);
```



## 批量插入后更新

```sql
INSERT INTO t (id, name) SELECT (random() * 10000000)::INT AS id, 'insert of update' FROM generate_series(1, 5000) t(i) ON CONFLICT (id) DO UPDATE SET name= EXCLUDED.name;

--                                                              QUERY PLAN

-- *------------------------------------------------------------------------------------------------------------------------------------*

-- Insert on t  (cost=0.00..137.50 rows=5000 width=520) (actual time=1122.742..1122.743 rows=0 loops=1)

-- Conflict Resolution: UPDATE

-- Conflict Arbiter Indexes: t_pkey

-- Tuples Inserted: 0

-- Conflicting Tuples: 5000

-- ->  Subquery Scan on "*SELECT*"  (cost=0.00..137.50 rows=5000 width=520) (actual time=0.848..4.896 rows=5000 loops=1)

-- ​         ->  Function Scan on generate_series t  (cost=0.00..87.50 rows=5000 width=36) (actual time=0.847..3.427 rows=5000 loops=1)

-- Planning Time: 0.066 ms

-- Execution Time: 1122.808 ms

-- (9 rows)

-- (END)
```

共耗时**1122.808 ms**



## 临时表批量更新

```sql
EXPLAIN ANALYZE INSERT INTO tmp_t (id, name) SELECT (random() * 10000000)::INT, 'update by tmp' FROM generate_series(1, 5000) t(i);
-- ​                                                             QUERY PLAN

-- *------------------------------------------------------------------------------------------------------------------------------------*
-- Insert on tmp_t  (cost=0.00..50.00 rows=5000 width=520) (actual time=12.552..12.553 rows=0 loops=1)
-- ->  Function Scan on generate_series t  (cost=0.00..50.00 rows=5000 width=520) (actual time=0.720..1.421 rows=5000 loops=1)
-- Planning Time: 0.071 ms
-- Execution Time: 12.647 ms
-- (4 rows)


EXPLAIN ANALYZE UPDATE t SET id = tmp.id, name = tmp.name FROM tmp_t tmp WHERE tmp.id=t.id;
--                                                                  QUERY PLAN
-- ---------------------------------------------------------------------------------------------------------------------------------------------

--  Update on t  (cost=17.24..413.21 rows=5000 width=30) (actual time=41.770..41.771 rows=0 loops=1)
--    ->  Merge Join  (cost=17.24..413.21 rows=5000 width=30) (actual time=0.086..10.119 rows=5000 loops=1)
--          Merge Cond: (t.id = tmp.id)
--          ->  Index Scan using t_pkey on t  (cost=0.43..343476.23 rows=10019520 width=10) (actual time=0.051..5.242 rows=5001 loops=1)
--          ->  Index Scan using tmp_t_pkey on tmp_t tmp  (cost=0.28..174.28 rows=5000 width=24) (actual time=0.032..1.749 rows=5000 loops=1)
--  Planning Time: 0.907 ms
--  Execution Time: 46.154 ms
-- (7 rows)
```

共耗时**58.801 ms**



相比之下*临时表批量更新* 比*批量插入后更新*执行效率要**快近20倍**。

当然*临时表批量更新*也有缺点：

- 实现更复杂
- 需要同时维护两个表
- 临时表会产生冗余数据

