# Задание 1: BRIN индексы и bitmap-сканирование

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

4. Создайте BRIN индекс по колонке category:
   ```sql
   CREATE INDEX t_books_brin_cat_idx ON t_books USING brin(category);
   ```

5. Найдите книги с NULL значением category:
   ```sql
   EXPLAIN ANALYZE
   SELECT * FROM t_books WHERE category IS NULL;
   ```
   
   *План выполнения:*
   "Bitmap Heap Scan on t_books  (cost=12.00..16.01 rows=1 width=33) (actual time=0.019..0.019 rows=0 loops=1)"
   "  Recheck Cond: (category IS NULL)"
   "  ->  Bitmap Index Scan on t_books_brin_cat_idx  (cost=0.00..12.00 rows=1 width=0) (actual time=0.015..0.016 rows=0 loops=1)"
   "        Index Cond: (category IS NULL)"
   "Planning Time: 0.462 ms"
   "Execution Time: 0.060 ms"

   Без индекса: 
   "Seq Scan on t_books  (cost=0.00..2725.00 rows=1 width=33) (actual time=14.259..14.259 rows=0 loops=1)"
"  Filter: (category IS NULL)"
"  Rows Removed by Filter: 150000"
"Planning Time: 0.153 ms"
"Execution Time: 14.281 ms"
      
   *Объясните результат:*
   Использование BRIN индекса здесь ускоряет работу запроса, т.к. он позволяет сразу отмести те блоки, где нет значений NULL, и рассматривать последовательно только те, где они есть.

1. Создайте BRIN индекс по автору:
   ```sql
   CREATE INDEX t_books_brin_author_idx ON t_books USING brin(author);
   ```

2. Выполните поиск по категории и автору:
   ```sql
   EXPLAIN ANALYZE
   SELECT * FROM t_books 
   WHERE category = 'INDEX' AND author = 'SYSTEM';
   ```
   
   *План выполнения:*
```
   "Bitmap Heap Scan on t_books  (cost=12.15..2323.02 rows=1 width=33) (actual time=1.490..1.491 rows=0 loops=1)"
"  Recheck Cond: (((category)::text = 'INDEX'::text) AND ((author)::text = 'SYSTEM'::text))"
"  Rows Removed by Index Recheck: 8848"
"  Heap Blocks: lossy=73"
"  ->  Bitmap Index Scan on t_books_brin_cat_auth_idx  (cost=0.00..12.15 rows=72391 width=0) (actual time=0.057..0.058 rows=730 loops=1)"
"        Index Cond: (((category)::text = 'INDEX'::text) AND ((author)::text = 'SYSTEM'::text))"
"Planning Time: 0.242 ms"
"Execution Time: 1.519 ms"
```
   
   *Объясните результат (обратите внимание на bitmap scan):*
   ```
   он не сработал, индекс не использовался. т.к. данные не упорядоченные, и поэтому невозможно органично разбить все данные на блоки.
   ```

3. Получите список уникальных категорий:
   ```sql
   EXPLAIN ANALYZE
   SELECT DISTINCT category 
   FROM t_books 
   ORDER BY category;
   ```
   
   *План выполнения:*
   ```
   "  Sort Key: category"
   "  Sort Method: quicksort  Memory: 25kB"
   "  ->  HashAggregate  (cost=3100.00..3100.06 rows=6 width=7) (actual time=51.982..51.985 rows=6 loops=1)"
   "        Group Key: category"
   "        Batches: 1  Memory Usage: 24kB"
   "        ->  Seq Scan on t_books  (cost=0.00..2725.00 rows=150000 width=7) (actual time=0.006..13.204 rows=150000 loops=1)"
   "Planning Time: 0.112 ms"
   "Execution Time: 52.061 ms"`
```
   
   *Объясните результат:*
у нас есть только БРИН индекс на категории, который содержит мин-мах значения, поэтому для поиска уникальных значений он не подходит (т.к. нет той нужной информации в самом индексе, как в б-дереве например), все равно приходится обращаться к таблице, запрос не ускорился

4. Подсчитайте книги, где автор начинается на 'S':
   ```sql
   EXPLAIN ANALYZE
   SELECT COUNT(*) 
   FROM t_books 
   WHERE author LIKE 'S%';
   ```
   
   *План выполнения:*
   "Aggregate  (cost=3100.03..3100.05 rows=1 width=8) (actual time=23.661..23.663 rows=1 loops=1)"
"  ->  Seq Scan on t_books  (cost=0.00..3100.00 rows=14 width=0) (actual time=23.653..23.653 rows=0 loops=1)"
"        Filter: ((author)::text ~~ 'S%'::text)"
"        Rows Removed by Filter: 150000"
"Planning Time: 0.274 ms"
"Execution Time: 23.705 ms"
   
   *Объясните результат:*
   Брин индекс даже не использовался, он не эффективен для префискного поиска, т.к. хранит только мин-макс значения. т.е. если в блоке есть потенциально строки, начинающиеся на S, их может быть слишком много из-за неупорядоченности изначальных данных, что приведет к избыточному сканированию всех строк, так что планировщик решил просто последовательно все просканировать.

5.  Создайте индекс для регистронезависимого поиска:
    ```sql
    CREATE INDEX t_books_lower_title_idx ON t_books(LOWER(title));
    ```

6.  Подсчитайте книги, начинающиеся на 'O':
    ```sql
    EXPLAIN ANALYZE
    SELECT COUNT(*) 
    FROM t_books 
    WHERE LOWER(title) LIKE 'o%';
    ```
   
   *План выполнения:*
   ```
   "Aggregate  (cost=3476.88..3476.89 rows=1 width=8) (actual time=71.738..71.740 rows=1 loops=1)"
   "  ->  Seq Scan on t_books  (cost=0.00..3475.00 rows=750 width=0) (actual time=71.727..71.730 rows=1 loops=1)"
   "        Filter: (lower((title)::text) ~~ 'o%'::text)"
   "        Rows Removed by Filter: 149999"
   "Planning Time: 0.382 ms"
   "Execution Time: 71.785 ms"
   ```
   *Объясните результат:*
   индекс не использовался вообще, т.к. здесь помимо индекса б-дерева во время запроса используется функция (lower) которую нужно применить ко всем строкам, для того чтобы запустить поиск. поэтому это не позволяет индексу вступить в игру и чето ускорить

7.  Удалите созданные индексы:
    ```sql
    DROP INDEX t_books_brin_cat_idx;
    DROP INDEX t_books_brin_author_idx;
    DROP INDEX t_books_lower_title_idx;
    ```

8.  Создайте составной BRIN индекс:
    ```sql
    CREATE INDEX t_books_brin_cat_auth_idx ON t_books 
    USING brin(category, author);
    ```

9.  Повторите запрос из шага 7:
    ```sql
    EXPLAIN ANALYZE
    SELECT * FROM t_books 
    WHERE category = 'INDEX' AND author = 'SYSTEM';
    ```
   
   *План выполнения:*
   ```
"Bitmap Heap Scan on t_books  (cost=12.15..2323.02 rows=1 width=33) (actual time=2.097..2.098 rows=0 loops=1)"
"  Recheck Cond: (((category)::text = 'INDEX'::text) AND ((author)::text = 'SYSTEM'::text))"
"  Rows Removed by Index Recheck: 8848"
"  Heap Blocks: lossy=73"
"  ->  Bitmap Index Scan on t_books_brin_cat_auth_idx  (cost=0.00..12.15 rows=72391 width=0) (actual time=0.035..0.036 rows=730 loops=1)"
"        Index Cond: (((category)::text = 'INDEX'::text) AND ((author)::text = 'SYSTEM'::text))"
"Planning Time: 0.234 ms"
"Execution Time: 2.128 ms"
```
   
   *Объясните результат:*
   Индекс ускорил работу запроса, но не супер значительно (всего в 5 раз), т.к. брин индекс хранит пары мин-макс для каждого составного элемента (category, author), он не супер эффективно работает с запросами полного совпадения, лучше ускоряет запросы диапазонов.