## Задание 2

1. Удалите старую базу данных, если есть:
    ```shell
    docker compose down
    ```

2. Поднимите базу данных из src/docker-compose.yml:
    ```shell
    docker compose down && docker compose up -d
    ```

3. Обновите статистику:
    ```sql
    ANALYZE t_books;
    ```

4. Создайте полнотекстовый индекс:
    ```sql
    CREATE INDEX t_books_fts_idx ON t_books 
    USING GIN (to_tsvector('english', title));
    ```

5. Найдите книги, содержащие слово 'expert':
    ```sql
    EXPLAIN ANALYZE
    SELECT * FROM t_books 
    WHERE to_tsvector('english', title) @@ to_tsquery('english', 'expert');
    ```
    
    *План выполнения:*
"Bitmap Heap Scan on t_books  (cost=21.03..1336.08 rows=750 width=33) (actual time=0.046..0.047 rows=1 loops=1)"
"  Recheck Cond: (to_tsvector('english'::regconfig, (title)::text) @@ '''expert'''::tsquery)"
"  Heap Blocks: exact=1"
"  ->  Bitmap Index Scan on t_books_fts_idx  (cost=0.00..20.84 rows=750 width=0) (actual time=0.038..0.039 rows=1 loops=1)"
"        Index Cond: (to_tsvector('english'::regconfig, (title)::text) @@ '''expert'''::tsquery)"
"Planning Time: 0.557 ms"
"Execution Time: 0.081 ms"
    
    *Объясните результат:*
    Полнотекстовый индекс значительно ускорил запрос, т.к. запрос и расчитан на поиск слов в тексте. гуд джоб

6. Удалите индекс:
    ```sql
    DROP INDEX t_books_fts_idx;
    ```

7. Создайте таблицу lookup:
    ```sql
    CREATE TABLE t_lookup (
         item_key VARCHAR(10) NOT NULL,
         item_value VARCHAR(100)
    );
    ```

8. Добавьте первичный ключ:
    ```sql
    ALTER TABLE t_lookup 
    ADD CONSTRAINT t_lookup_pk PRIMARY KEY (item_key);
    ```

9. Заполните данными:
    ```sql
    INSERT INTO t_lookup 
    SELECT 
         LPAD(CAST(generate_series(1, 150000) AS TEXT), 10, '0'),
         'Value_' || generate_series(1, 150000);
    ```

10. Создайте кластеризованную таблицу:
     ```sql
     CREATE TABLE t_lookup_clustered (
          item_key VARCHAR(10) PRIMARY KEY,
          item_value VARCHAR(100)
     );
     ```

11. Заполните её теми же данными:
     ```sql
     INSERT INTO t_lookup_clustered 
     SELECT * FROM t_lookup;
     
     CLUSTER t_lookup_clustered USING t_lookup_clustered_pkey;
     ```

12. Обновите статистику:
     ```sql
     ANALYZE t_lookup;
     ANALYZE t_lookup_clustered;
     ```

13. Выполните поиск по ключу в обычной таблице:
     ```sql
     EXPLAIN ANALYZE
     SELECT * FROM t_lookup WHERE item_key = '0000000455';
     ```
     
     *План выполнения:*
"Index Scan using t_lookup_pk on t_lookup  (cost=0.42..8.44 rows=1 width=23) (actual time=0.031..0.083 rows=1 loops=1)"
"  Index Cond: ((item_key)::text = '0000000455'::text)"
"Planning Time: 0.158 ms"
"Execution Time: 0.102 ms"
     
     *Объясните результат:*
     item key является первичным ключом, поэтому для него был автоматически создан б-три индекс, запрос оказался им ускорен

14. Выполните поиск по ключу в кластеризованной таблице:
     ```sql
     EXPLAIN ANALYZE
     SELECT * FROM t_lookup_clustered WHERE item_key = '0000000455';
     ```
     
     *План выполнения:*
"Index Scan using t_lookup_clustered_pkey on t_lookup_clustered  (cost=0.42..8.44 rows=1 width=23) (actual time=0.058..0.059 rows=1 loops=1)"
"  Index Cond: ((item_key)::text = '0000000455'::text)"
"Planning Time: 0.165 ms"
"Execution Time: 0.087 ms"
     
     *Объясните результат:*
     В кластеризованной таблице item key так же является PK, поэтому при реализации запроса так же используется индекс, но за счет кластеризации он работает немного быстрее

15. Создайте индекс по значению для обычной таблицы:
     ```sql
     CREATE INDEX t_lookup_value_idx ON t_lookup(item_value);
     ```

16. Создайте индекс по значению для кластеризованной таблицы:
     ```sql
     CREATE INDEX t_lookup_clustered_value_idx 
     ON t_lookup_clustered(item_value);
     ```

17. Выполните поиск по значению в обычной таблице:
     ```sql
     EXPLAIN ANALYZE
     SELECT * FROM t_lookup WHERE item_value = 'T_BOOKS';
     ```
     
     *План выполнения:*
"Index Scan using t_lookup_value_idx on t_lookup  (cost=0.42..8.44 rows=1 width=23) (actual time=0.035..0.035 rows=0 loops=1)"
"  Index Cond: ((item_value)::text = 'T_BOOKS'::text)"
"Planning Time: 0.306 ms"
"Execution Time: 0.053 ms"
     
     *Объясните результат:*
     Индекс здорово ускорил запрос, т.к. поиск точного значения среди сравниваемых строк, гуд джоб

18. Выполните поиск по значению в кластеризованной таблице:
     ```sql
     EXPLAIN ANALYZE
     SELECT * FROM t_lookup_clustered WHERE item_value = 'T_BOOKS';
     ```
     
     *План выполнения:*
"Index Scan using t_lookup_clustered_value_idx on t_lookup_clustered  (cost=0.42..8.44 rows=1 width=23) (actual time=0.036..0.036 rows=0 loops=1)"
"  Index Cond: ((item_value)::text = 'T_BOOKS'::text)"
"Planning Time: 0.353 ms"
"Execution Time: 0.068 ms"
     
     *Объясните результат:*
     Тут он тоже ускорил запрос, но все-таки работает чуть-чуть медленнее, но незначительно

19. Сравните производительность поиска по значению в обычной и кластеризованной таблицах:
     
     *Сравнение:*
     В кластеризованной таблице производительность работы поиска по значению немного ниже, чем в некластеризованной таблице, но незначительно. Изначальная таблица очень упорядочена из-за того, как она создавалась. На менее упорядоченных таблицах кластеризация оказала бы большее влияние на ускорение/замедление запросов.