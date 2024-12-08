# Задание 2: Специальные случаи использования индексов

# Партиционирование и специальные случаи использования индексов

1. Удалите прошлый инстанс PostgreSQL - `docker-compose down` в папке `src` и запустите новый: `docker-compose up -d`.

2. Создайте партиционированную таблицу и заполните её данными:

    ```sql
    -- Создание партиционированной таблицы
    CREATE TABLE t_books_part (
        book_id     INTEGER      NOT NULL,
        title       VARCHAR(100) NOT NULL,
        category    VARCHAR(30),
        author      VARCHAR(100) NOT NULL,
        is_active   BOOLEAN      NOT NULL
    ) PARTITION BY RANGE (book_id);

    -- Создание партиций
    CREATE TABLE t_books_part_1 PARTITION OF t_books_part
        FOR VALUES FROM (MINVALUE) TO (50000);

    CREATE TABLE t_books_part_2 PARTITION OF t_books_part
        FOR VALUES FROM (50000) TO (100000);

    CREATE TABLE t_books_part_3 PARTITION OF t_books_part
        FOR VALUES FROM (100000) TO (MAXVALUE);

    -- Копирование данных из t_books
    INSERT INTO t_books_part 
    SELECT * FROM t_books;
    ```

3. Обновите статистику таблиц:
   ```sql
   ANALYZE t_books;
   ANALYZE t_books_part;
   ```
   
   *Результат:*
```
workshop.public> ANALYZE t_books
[2024-12-08 01:31:21] completed in 141 ms
workshop.public> ANALYZE t_books_part
[2024-12-08 01:31:25] completed in 4 ms
```

4. Выполните запрос для поиска книги с id = 18:
   ```sql
   EXPLAIN ANALYZE
   SELECT * FROM t_books_part WHERE book_id = 18;
   ```
   
   *План выполнения:*

| QUERY PLAN |
| :--- |
| Index Scan using t\_books\_part\_1\_book\_id\_idx on t\_books\_part\_1 t\_books\_part  \(cost=0.29..8.31 rows=1 width=32\) \(actual time=0.010..0.011 rows=1 loops=1\) |
|   Index Cond: \(book\_id = 18\) |
| Planning Time: 0.209 ms |
| Execution Time: 0.027 ms |


   
   *Объясните результат:*
   Запрос выполнился очень быстро за счет использования партиционированной таблицы. 
   Сканирование прошло только в первой партиции.

5. Выполните поиск по названию книги:
   ```sql
   EXPLAIN ANALYZE
   SELECT * FROM t_books_part 
   WHERE title = 'Expert PostgreSQL Architecture';
   ```
   
   *План выполнения:*

| QUERY PLAN |
| :--- |
| Append  \(cost=0.00..3101.05 rows=3 width=33\) \(actual time=2.931..12.692 rows=1 loops=1\) |
|   -&gt;  Seq Scan on t\_books\_part\_1  \(cost=0.00..1032.99 rows=1 width=32\) \(actual time=2.930..2.932 rows=1 loops=1\) |
|         Filter: \(\(title\)::text = 'Expert PostgreSQL Architecture'::text\) |
|         Rows Removed by Filter: 49998 |
|   -&gt;  Seq Scan on t\_books\_part\_2  \(cost=0.00..1034.00 rows=1 width=33\) \(actual time=4.871..4.871 rows=0 loops=1\) |
|         Filter: \(\(title\)::text = 'Expert PostgreSQL Architecture'::text\) |
|         Rows Removed by Filter: 50000 |
|   -&gt;  Seq Scan on t\_books\_part\_3  \(cost=0.00..1034.05 rows=1 width=34\) \(actual time=4.883..4.883 rows=0 loops=1\) |
|         Filter: \(\(title\)::text = 'Expert PostgreSQL Architecture'::text\) |
|         Rows Removed by Filter: 50004 |
| Planning Time: 0.283 ms |
| Execution Time: 12.744 ms |


   
   *Объясните результат:*
   Здесь сканирование прошло последовательно по всем партициям.

6. Создайте партиционированный индекс:
   ```sql
   CREATE INDEX ON t_books_part(title);
   ```
   
   *Результат:*
```
workshop.public> CREATE INDEX ON t_books_part(title)
[2024-12-08 01:35:07] completed in 17 ms
```

7. Повторите запрос из шага 5:
   ```sql
   EXPLAIN ANALYZE
   SELECT * FROM t_books_part 
   WHERE title = 'Expert PostgreSQL Architecture';
   ```
   
   *План выполнения:*

| QUERY PLAN |
| :--- |
| Append  \(cost=0.29..24.94 rows=3 width=33\) \(actual time=0.068..0.121 rows=1 loops=1\) |
|   -&gt;  Index Scan using t\_books\_part\_1\_title\_idx on t\_books\_part\_1  \(cost=0.29..8.31 rows=1 width=32\) \(actual time=0.068..0.068 rows=1 loops=1\) |
|         Index Cond: \(\(title\)::text = 'Expert PostgreSQL Architecture'::text\) |
|   -&gt;  Index Scan using t\_books\_part\_2\_title\_idx on t\_books\_part\_2  \(cost=0.29..8.31 rows=1 width=33\) \(actual time=0.031..0.031 rows=0 loops=1\) |
|         Index Cond: \(\(title\)::text = 'Expert PostgreSQL Architecture'::text\) |
|   -&gt;  Index Scan using t\_books\_part\_3\_title\_idx on t\_books\_part\_3  \(cost=0.29..8.31 rows=1 width=34\) \(actual time=0.019..0.019 rows=0 loops=1\) |
|         Index Cond: \(\(title\)::text = 'Expert PostgreSQL Architecture'::text\) |
| Planning Time: 0.774 ms |
| Execution Time: 0.151 ms |


   
   *Объясните результат:*
Здесь сканирование прошло по всем партициям, но с использованием индекса, что уменьшило время выполнения запроса в разы.

8. Удалите созданный индекс:
   ```sql
   DROP INDEX t_books_part_title_idx;
   ```
   
   *Результат:*

```
workshop.public> DROP INDEX t_books_part_title_idx
[2024-12-08 01:41:42] completed in 38 ms
```

9. Создайте индекс для каждой партиции:
   ```sql
   CREATE INDEX ON t_books_part_1(title);
   CREATE INDEX ON t_books_part_2(title);
   CREATE INDEX ON t_books_part_3(title);
   ```
   
   *Результат:*
```
workshop.public> CREATE INDEX ON t_books_part_1(title)
[2024-12-08 01:42:25] completed in 18 ms
workshop.public> CREATE INDEX ON t_books_part_2(title)
[2024-12-08 01:42:26] completed in 16 ms
workshop.public> CREATE INDEX ON t_books_part_3(title)
[2024-12-08 01:42:27] completed in 16 ms
```

10. Повторите запрос из шага 5:
    ```sql
    EXPLAIN ANALYZE
    SELECT * FROM t_books_part 
    WHERE title = 'Expert PostgreSQL Architecture';
    ```
    
    *План выполнения:*

| QUERY PLAN |
| :--- |
| Append  \(cost=0.29..24.94 rows=3 width=33\) \(actual time=0.211..0.354 rows=1 loops=1\) |
|   -&gt;  Index Scan using t\_books\_part\_1\_title\_idx on t\_books\_part\_1  \(cost=0.29..8.31 rows=1 width=32\) \(actual time=0.210..0.212 rows=1 loops=1\) |
|         Index Cond: \(\(title\)::text = 'Expert PostgreSQL Architecture'::text\) |
|   -&gt;  Index Scan using t\_books\_part\_2\_title\_idx on t\_books\_part\_2  \(cost=0.29..8.31 rows=1 width=33\) \(actual time=0.049..0.049 rows=0 loops=1\) |
|         Index Cond: \(\(title\)::text = 'Expert PostgreSQL Architecture'::text\) |
|   -&gt;  Index Scan using t\_books\_part\_3\_title\_idx on t\_books\_part\_3  \(cost=0.29..8.31 rows=1 width=34\) \(actual time=0.089..0.089 rows=0 loops=1\) |
|         Index Cond: \(\(title\)::text = 'Expert PostgreSQL Architecture'::text\) |
| Planning Time: 10.455 ms |
| Execution Time: 0.434 ms |


    
*Объясните результат:*
Время на планирование увеличилось, но это не будет иметь значения при больщом количечстве данных. 
Использование индексов для партиций уменьшило время выполнения запроса.

11. Удалите созданные индексы:
    ```sql
    DROP INDEX t_books_part_1_title_idx;
    DROP INDEX t_books_part_2_title_idx;
    DROP INDEX t_books_part_3_title_idx;
    ```
    
    *Результат:*

```
workshop.public> DROP INDEX t_books_part_1_title_idx
[2024-12-08 01:45:33] completed in 20 ms
workshop.public> DROP INDEX t_books_part_2_title_idx
[2024-12-08 01:45:34] completed in 15 ms
workshop.public> DROP INDEX t_books_part_3_title_idx
[2024-12-08 01:45:35] completed in 8 ms
```

12. Создайте обычный индекс по book_id:
    ```sql
    CREATE INDEX t_books_part_idx ON t_books_part(book_id);
    ```
    
    *Результат:*

```
workshop.public> CREATE INDEX t_books_part_idx ON t_books_part(book_id)
[2024-12-08 01:46:10] completed in 9 ms
```

13. Выполните поиск по book_id:
    ```sql
    EXPLAIN ANALYZE
    SELECT * FROM t_books_part WHERE book_id = 11011;
    ```
    
    *План выполнения:*

| QUERY PLAN |
| :--- |
| Index Scan using t\_books\_part\_1\_book\_id\_idx on t\_books\_part\_1 t\_books\_part  \(cost=0.29..8.31 rows=1 width=32\) \(actual time=0.012..0.012 rows=1 loops=1\) |
|   Index Cond: \(book\_id = 11011\) |
| Planning Time: 0.232 ms |
| Execution Time: 0.028 ms |

    
*Объясните результат:*

Сканирование прошло только в первой партиции, использование индекса привело к ускорению выполнения запроса. 

14. Создайте индекс по полю is_active:
    ```sql
    CREATE INDEX t_books_active_idx ON t_books(is_active);
    ```
    
    *Результат:*

```
workshop.public> CREATE INDEX t_books_active_idx ON t_books(is_active)
[2024-12-08 01:53:55] completed in 102 ms
```

15. Выполните поиск активных книг с отключенным последовательным сканированием:
    ```sql
    SET enable_seqscan = off;
    EXPLAIN ANALYZE
    SELECT * FROM t_books WHERE is_active = true;
    SET enable_seqscan = on;
    ```
    
    *План выполнения:*

| QUERY PLAN |
| :--- |
| Bitmap Heap Scan on t\_books  \(cost=838.53..2808.64 rows=74611 width=33\) \(actual time=1.747..8.478 rows=74925 loops=1\) |
|   Recheck Cond: is\_active |
|   Heap Blocks: exact=1224 |
|   -&gt;  Bitmap Index Scan on t\_books\_active\_idx  \(cost=0.00..819.88 rows=74611 width=0\) \(actual time=1.592..1.593 rows=74925 loops=1\) |
|         Index Cond: \(is\_active = true\) |
| Planning Time: 0.067 ms |
| Execution Time: 10.497 ms |


    
*Объясните результат:*
SET enable_seqscan = off позволяет принудительно использовать индексное сканирование по бит мэпам.

16. Создайте составной индекс:
    ```sql
    CREATE INDEX t_books_author_title_index ON t_books(author, title);
    ```
    
    *Результат:*

```
workshop.public> CREATE INDEX t_books_author_title_index ON t_books(author, title)
[2024-12-08 02:07:46] completed in 289 ms
```

17. Найдите максимальное название для каждого автора:
    ```sql
    EXPLAIN ANALYZE
    SELECT author, MAX(title) 
    FROM t_books 
    GROUP BY author;
    ```
    
    *План выполнения:*

| QUERY PLAN |
| :--- |
| HashAggregate  \(cost=3474.05..3484.05 rows=1000 width=42\) \(actual time=40.540..40.646 rows=1006 loops=1\) |
|   Group Key: author |
|   Batches: 1  Memory Usage: 193kB |
|   -&gt;  Seq Scan on t\_books  \(cost=0.00..2724.03 rows=150003 width=21\) \(actual time=0.010..6.922 rows=150003 loops=1\) |
| Planning Time: 0.089 ms |
| Execution Time: 40.776 ms |



    
*Объясните результат:*
Запрос выполнился с помощью хеш агрегации и последовательного сканирования.

18. Выберите первых 10 авторов:
    ```sql
    EXPLAIN ANALYZE
    SELECT DISTINCT author 
    FROM t_books 
    ORDER BY author 
    LIMIT 10;
    ```
    
    *План выполнения:*

| QUERY PLAN |
| :--- |
| Limit  \(cost=0.42..56.67 rows=10 width=10\) \(actual time=0.022..0.172 rows=10 loops=1\) |
|   -&gt;  Result  \(cost=0.42..5625.47 rows=1000 width=10\) \(actual time=0.022..0.171 rows=10 loops=1\) |
|         -&gt;  Unique  \(cost=0.42..5625.47 rows=1000 width=10\) \(actual time=0.021..0.170 rows=10 loops=1\) |
|               -&gt;  Index Only Scan using t\_books\_author\_title\_index on t\_books  \(cost=0.42..5250.46 rows=150003 width=10\) \(actual time=0.020..0.107 rows=1058 loops=1\) |
|                     Heap Fetches: 4 |
| Planning Time: 0.079 ms |
| Execution Time: 0.192 ms |


    
*Объясните результат:*
Запрос использовал индекс эффективно.

19. Выполните поиск и сортировку:
    ```sql
    EXPLAIN ANALYZE
    SELECT author, title 
    FROM t_books 
    WHERE author LIKE 'T%'
    ORDER BY author, title;
    ```
    
    *План выполнения:*

| QUERY PLAN |
| :--- |
| Sort  \(cost=3099.33..3099.37 rows=15 width=21\) \(actual time=11.429..11.431 rows=1 loops=1\) |
|   Sort Key: author, title |
|   Sort Method: quicksort  Memory: 25kB |
|   -&gt;  Seq Scan on t\_books  \(cost=0.00..3099.04 rows=15 width=21\) \(actual time=11.409..11.412 rows=1 loops=1\) |
|         Filter: \(\(author\)::text \~\~ 'T%'::text\) |
|         Rows Removed by Filter: 150002 |
| Planning Time: 0.210 ms |
| Execution Time: 11.453 ms |
 


*Объясните результат:*
Было выполнено последовательное сканирование изза LIKE 'T%', что сказалось на скорости выполнения запроса.

20. Добавьте новую книгу:
    ```sql
    INSERT INTO t_books (book_id, title, author, category, is_active)
    VALUES (150001, 'Cookbook', 'Mr. Hide', NULL, true);
    COMMIT;
    ```
    
    *Результат:*

```
workshop.public> INSERT INTO t_books (book_id, title, author, category, is_active)
                 VALUES (150001, 'Cookbook', 'Mr. Hide', NULL, true)
[2024-12-08 02:18:19] 1 row affected in 27 ms
workshop.public> COMMIT
[2024-12-08 02:18:24] [25P01] there is no transaction in progress
[2024-12-08 02:18:24] completed in 18 ms
```

21. Создайте индекс по категории:
    ```sql
    CREATE INDEX t_books_cat_idx ON t_books(category);
    ```
    
    *Результат:*

```
workshop.public> CREATE INDEX t_books_cat_idx ON t_books(category)
[2024-12-08 02:19:18] completed in 154 ms
```

22. Найдите книги без категории:
    ```sql
    EXPLAIN ANALYZE
    SELECT author, title 
    FROM t_books 
    WHERE category IS NULL;
    ```
    
    *План выполнения:*

| QUERY PLAN |
| :--- |
| Index Scan using t\_books\_cat\_null\_idx on t\_books  \(cost=0.12..7.98 rows=1 width=21\) \(actual time=0.021..0.021 rows=1 loops=1\) |
| Planning Time: 0.308 ms |
| Execution Time: 0.033 ms |


    
*Объясните результат:*
Быстрое выполнение запроса с помощью индекса.

23. Создайте частичные индексы:
    ```sql
    DROP INDEX t_books_cat_idx;
    CREATE INDEX t_books_cat_null_idx ON t_books(category) WHERE category IS NULL;
    ```
    
    *Результат:*

```
workshop.public> DROP INDEX t_books_cat_idx
[2024-12-08 02:46:21] completed in 11 ms
workshop.public> CREATE INDEX t_books_cat_null_idx ON t_books(category) WHERE category IS NULL
[2024-12-08 02:46:33] completed in 32 ms
```

24. Повторите запрос из шага 22:
    ```sql
    EXPLAIN ANALYZE
    SELECT author, title 
    FROM t_books 
    WHERE category IS NULL;
    ```
    
    *План выполнения:*

| QUERY PLAN |
| :--- |
| Index Scan using t\_books\_cat\_null\_idx on t\_books  \(cost=0.12..7.98 rows=1 width=21\) \(actual time=0.065..0.067 rows=1 loops=1\) |
| Planning Time: 0.372 ms |
| Execution Time: 0.090 ms |


    
*Объясните результат:*
Снова оптимизированное выполнение запроса с помощью индекса с дополнительным условием.

25. Создайте частичный уникальный индекс:
    ```sql
    CREATE UNIQUE INDEX t_books_selective_unique_idx 
    ON t_books(title) 
    WHERE category = 'Science';
    
    -- Протестируйте его
    INSERT INTO t_books (book_id, title, author, category, is_active)
    VALUES (150002, 'Unique Science Book', 'Author 1', 'Science', true);
    
    -- Попробуйте вставить дубликат
    INSERT INTO t_books (book_id, title, author, category, is_active)
    VALUES (150003, 'Unique Science Book', 'Author 2', 'Science', true);
    
    -- Но можно вставить такое же название для другой категории
    INSERT INTO t_books (book_id, title, author, category, is_active)
    VALUES (150004, 'Unique Science Book', 'Author 3', 'History', true);
    ```
    
    *Результат:*

```
workshop.public> CREATE UNIQUE INDEX t_books_selective_unique_idx
                     ON t_books(title)
                     WHERE category = 'Science'
[2024-12-08 02:47:42] completed in 54 ms
workshop.public> INSERT INTO t_books (book_id, title, author, category, is_active)
                 VALUES (150002, 'Unique Science Book', 'Author 1', 'Science', true)
[2024-12-08 02:47:58] 1 row affected in 17 ms
workshop.public> INSERT INTO t_books (book_id, title, author, category, is_active)
                 VALUES (150003, 'Unique Science Book', 'Author 2', 'Science', true)
[2024-12-08 02:48:09] [23505] ERROR: duplicate key value violates unique constraint "t_books_selective_unique_idx"
[2024-12-08 02:48:09] Detail: Key (title)=(Unique Science Book) already exists.
workshop.public> INSERT INTO t_books (book_id, title, author, category, is_active)
                 VALUES (150004, 'Unique Science Book', 'Author 3', 'History', true)
[2024-12-08 02:48:15] 1 row affected in 16 ms
```
    
*Объясните результат:*
Частичный уникальный индекс позволяет гарантировать уникальность значений для категории 'Science'.
