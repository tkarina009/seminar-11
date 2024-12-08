## Задание 3

1. Создайте таблицу с большим количеством данных:
    ```sql
    CREATE TABLE test_cluster AS 
    SELECT 
        generate_series(1,1000000) as id,
        CASE WHEN random() < 0.5 THEN 'A' ELSE 'B' END as category,
        md5(random()::text) as data;
    ```

2. Создайте индекс:
    ```sql
    CREATE INDEX test_cluster_cat_idx ON test_cluster(category);
    ```

3. Измерьте производительность до кластеризации:
    ```sql
    EXPLAIN ANALYZE
    SELECT * FROM test_cluster WHERE category = 'A';
    ```
    
    *План выполнения:*
"Bitmap Heap Scan on test_cluster  (cost=59.17..7696.73 rows=5000 width=68) (actual time=26.765..199.419 rows=501461 loops=1)"
"  Recheck Cond: (category = 'A'::text)"
"  Heap Blocks: exact=8334"
"  ->  Bitmap Index Scan on test_cluster_cat_idx  (cost=0.00..57.92 rows=5000 width=0) (actual time=24.562..24.562 rows=501461 loops=1)"
"        Index Cond: (category = 'A'::text)"
"Planning Time: 0.214 ms"
"Execution Time: 233.205 ms"
    
    *Объясните результат:*
    проходимся по данным и ищем те, которые удовлетворяют условию, т.к. данных много это достаточно долго

1. Выполните кластеризацию:
    ```sql
    CLUSTER test_cluster USING test_cluster_cat_idx;
    ```
    
    *Результат:*
CLUSTER

Query returned successfully in 1 secs 669 msec.

5. Измерьте производительность после кластеризации:
    ```sql
    EXPLAIN ANALYZE
    SELECT * FROM test_cluster WHERE category = 'A';
    ```
    
    *План выполнения:*
"Bitmap Heap Scan on test_cluster  (cost=5605.38..20224.79 rows=502833 width=39) (actual time=28.243..125.884 rows=501461 loops=1)"
"  Recheck Cond: (category = 'A'::text)"
"  Heap Blocks: exact=4179"
"  ->  Bitmap Index Scan on test_cluster_cat_idx  (cost=0.00..5479.67 rows=502833 width=0) (actual time=27.400..27.400 rows=501461 loops=1)"
"        Index Cond: (category = 'A'::text)"
"Planning Time: 0.532 ms"
"Execution Time: 151.517 ms"
    
    *Объясните результат:*
    Таблица упорядочилась в соответствии с индексом по категории, таким образом наш запрос, связанный с категорией, оказался ускорен

1. Сравните производительность до и после кластеризации:
    
    *Сравнение:*
    производительность после кластеризации увеличилась, т.к. теперь данные отсортированы что позволяет быстрее выполнять запросы