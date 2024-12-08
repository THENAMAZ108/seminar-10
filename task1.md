# Задание 1. B-tree индексы в PostgreSQL

1. Запустите БД через docker compose в ./src/docker-compose.yml:

2. Выполните запрос для поиска книги с названием 'Oracle Core' и получите план выполнения:
   ```sql
   EXPLAIN ANALYZE
   SELECT * FROM t_books WHERE title = 'Oracle Core';
   ```
   
   *План выполнения:*

```
Seq Scan on t_books  (cost=0.00..3100.00 rows=1 width=33) (actual time=8.803..8.806 rows=1 loops=1)
  Filter: ((title)::text = 'Oracle Core'::text)
  Rows Removed by Filter: 149999
Planning Time: 0.056 ms
Execution Time: 8.821 ms
```   
   *Объясните результат:*
   План выполнения запроса показал, что сканирование таблицы = 8.8 мс и исключило 149999 строк, чтобы найти одну нужную строку. Это неэффективно для больших таблиц.

3. Создайте B-tree индексы:
   ```sql
   CREATE INDEX t_books_title_idx ON t_books(title);
   CREATE INDEX t_books_active_idx ON t_books(is_active);
   ```
   
   *Результат:*
   [Вставьте результат выполнения]

4. Проверьте информацию о созданных индексах:
   ```sql
   SELECT schemaname, tablename, indexname, indexdef
   FROM pg_catalog.pg_indexes
   WHERE tablename = 't_books';
   ```
   
   *Результат:*
    ```
    public,t_books,t_books_id_pk,CREATE UNIQUE INDEX t_books_id_pk ON public.t_books USING btree (book_id)
    public,t_books,t_books_title_idx,CREATE INDEX t_books_title_idx ON public.t_books USING btree (title)
    public,t_books,t_books_active_idx,CREATE INDEX t_books_active_idx ON public.t_books USING btree (is_active)
    ```
   
   *Объясните результат:*
   Эти данные подтверждают успешное создание индексов в таблице t_books.


5. Обновите статистику таблицы:
   ```sql
   ANALYZE t_books;
   ```
   
   *Результат:*
   ```
    workshop.public> ANALYZE t_books
    [2024-12-07 23:58:08] completed in 131 ms
   ```

6. Выполните запрос для поиска книги 'Oracle Core' и получите план выполнения:
   ```sql
   EXPLAIN ANALYZE
   SELECT * FROM t_books WHERE title = 'Oracle Core';
   ```
   
   *План выполнения:*
    ```
    Index Scan using t_books_title_idx on t_books  (cost=0.42..8.44 rows=1 width=33) (actual time=0.038..0.039 rows=1 loops=1)
      Index Cond: ((title)::text = 'Oracle Core'::text)
    Planning Time: 0.326 ms
    Execution Time: 0.060 ms
    ```
   
   *Объясните результат:*
   - Стоимость выполнения запроса значительно уменьшилась по сравнению с предыдущим планом.
   - Время выполнения запроса значительно сократилось до 0.039 мс. 
   - Время на планирование запроса немного увеличилось, но это незначительно по сравнению с улучшением времени выполнения.


7. Выполните запрос для поиска книги по book_id и получите план выполнения:
   ```sql
   EXPLAIN ANALYZE
   SELECT * FROM t_books WHERE book_id = 18;
   ```
   
   *План выполнения:*
    ```
    Index Scan using t_books_id_pk on t_books  (cost=0.42..8.44 rows=1 width=33) (actual time=0.412..0.415 rows=1 loops=1)
    Index Cond: (book_id = 18)
    Planning Time: 0.231 ms
    Execution Time: 0.469 ms
    ```
   
   *Объясните результат:*
Для первичного ключа автоматически создается индекс, поэтому поиск происходит быстрее.


8. Выполните запрос для поиска активных книг и получите план выполнения:
   ```sql
   EXPLAIN ANALYZE
   SELECT * FROM t_books WHERE is_active = true;
   ```
   
   *План выполнения:*
   ```
    Seq Scan on t_books  (cost=0.00..2725.00 rows=74725 width=33) (actual time=0.030..13.520 rows=74809 loops=1)
    Filter: is_active
    Rows Removed by Filter: 75191
    Planning Time: 0.118 ms
    Execution Time: 15.474 ms
    ```
   
   *Объясните результат:*
Несмотря на созданный индекс, PostgreSQL выбрал последовательное сканирование, так как это быстрее для boolean.

9. Посчитайте количество строк и уникальных значений:
   ```sql
   SELECT 
       COUNT(*) as total_rows,
       COUNT(DISTINCT title) as unique_titles,
       COUNT(DISTINCT category) as unique_categories,
       COUNT(DISTINCT author) as unique_authors
   FROM t_books;
   ```
   
   *Результат:*

   | total\_rows | unique\_titles | unique\_categories | unique\_authors |
   | :--- | :--- | :--- | :--- |
   | 150000 | 150000 | 6 | 1003 |



10. Удалите созданные индексы:
    ```sql
    DROP INDEX t_books_title_idx;
    DROP INDEX t_books_active_idx;
    ```
    
    *Результат:*
    ```
    workshop.public> DROP INDEX t_books_title_idx
    [2024-12-08 00:34:42] completed in 17 ms
    workshop.public> DROP INDEX t_books_active_idx
    [2024-12-08 00:34:42] completed in 8 ms```

11. Основываясь на предыдущих результатах, создайте индексы для оптимизации следующих запросов:
    a. `WHERE title = $1 AND category = $2`
    b. `WHERE title = $1`
    c. `WHERE category = $1 AND author = $2`
    d. `WHERE author = $1 AND book_id = $2`
    
    *Созданные индексы:*
    ```sql
    CREATE INDEX t_books_title_category_idx ON t_books(title, category);
    CREATE INDEX t_books_title_idx ON t_books(title);
    CREATE INDEX t_books_category_author_idx ON t_books(category, author);
    CREATE INDEX t_books_author_book_id_idx ON t_books(author, book_id);
    ```
    
    *Объясните ваше решение:*
    Для составных запросов, таких как title и category, составные индексы позволяют эффективно фильтровать данные по нескольким столбцам одновременно.


12. Протестируйте созданные индексы.
    
    *Результаты тестов:*

a

| QUERY PLAN |
| :--- |
| Index Scan using t\_books\_category\_author\_idx on t\_books  \(cost=0.29..8.31 rows=1 width=33\) \(actual time=0.083..0.085 rows=0 loops=1\) |
|   Index Cond: \(\(category\)::text = 'Some Category'::text\) |
|   Filter: \(\(title\)::text = 'Some Title'::text\) |
| Planning Time: 0.346 ms |
| Execution Time: 0.166 ms |

b

| QUERY PLAN |
| :--- |
| Index Scan using t\_books\_title\_idx on t\_books  \(cost=0.42..8.44 rows=1 width=33\) \(actual time=0.023..0.023 rows=0 loops=1\) |
|   Index Cond: \(\(title\)::text = 'Some Title'::text\) |
| Planning Time: 0.069 ms |
| Execution Time: 0.050 ms |

c

| QUERY PLAN |
| :--- |
| Index Scan using t\_books\_category\_author\_idx on t\_books  \(cost=0.29..8.31 rows=1 width=33\) \(actual time=0.027..0.028 rows=0 loops=1\) |
|   Index Cond: \(\(\(category\)::text = 'Some Category'::text\) AND \(\(author\)::text = 'Some Author'::text\)\) |
| Planning Time: 0.108 ms |
| Execution Time: 0.058 ms |

d

| QUERY PLAN |
| :--- |
| Index Scan using t\_books\_author\_book\_id\_idx on t\_books  \(cost=0.42..8.44 rows=1 width=33\) \(actual time=0.275..0.276 rows=0 loops=1\) |
|   Index Cond: \(\(\(author\)::text = 'Some Author'::text\) AND \(book\_id = 18\)\) |
| Planning Time: 0.200 ms |
| Execution Time: 0.300 ms |


*Объясните результаты:*
    Все индексы работают эффективно. Стоимость запросов, время выполнения уменьшились. Время планирование слегка увеличилось.


13. Выполните регистронезависимый поиск по началу названия:
    ```sql
    EXPLAIN ANALYZE
    SELECT * FROM t_books WHERE title ILIKE 'Relational%';
    ```
    
    *План выполнения:*

| QUERY PLAN |
| :--- |
| Seq Scan on t\_books  \(cost=0.00..3100.00 rows=15 width=33\) \(actual time=79.473..79.474 rows=0 loops=1\) |
|   Filter: \(\(title\)::text \~\~\* 'Relational%'::text\) |
|   Rows Removed by Filter: 150000 |
| Planning Time: 0.695 ms |
| Execution Time: 79.488 ms |


*Объясните результат:*
Последовательное сканирование таблицы. Видимо нет поддержки регистронезависимого поиска.


14. Создайте функциональный индекс:
    ```sql
    CREATE INDEX t_books_up_title_idx ON t_books(UPPER(title));
    ```
    
    *Результат:*
    ```
    workshop.public> CREATE INDEX t_books_up_title_idx ON t_books(UPPER(title))
    [2024-12-08 00:53:52] completed in 235 ms
    ```

15. Выполните запрос из шага 13 с использованием UPPER:
    ```sql
    EXPLAIN ANALYZE
    SELECT * FROM t_books WHERE UPPER(title) LIKE 'RELATIONAL%';
    ```
    
    *План выполнения:*

| QUERY PLAN |
| :--- |
| Seq Scan on t\_books  \(cost=0.00..3475.00 rows=750 width=33\) \(actual time=39.804..39.804 rows=0 loops=1\) |
|   Filter: \(upper\(\(title\)::text\) \~\~ 'RELATIONAL%'::text\) |
|   Rows Removed by Filter: 150000 |
| Planning Time: 0.817 ms |
| Execution Time: 39.825 ms |

    
*Объясните результат:*

Время выполнения запроса с использованием функции UPPER уменьшилось по сравнению с ILIKE, 
но все еще относительно высоко из-за использования последовательного сканирования. 
Индекс UPPER(title) не был использован эффективно

16. Выполните поиск подстроки:
    ```sql
    EXPLAIN ANALYZE
    SELECT * FROM t_books WHERE title ILIKE '%Core%';
    ```
    
    *План выполнения:*

| QUERY PLAN |
| :--- |
| Seq Scan on t\_books  \(cost=0.00..3100.00 rows=15 width=33\) \(actual time=68.758..68.762 rows=1 loops=1\) |
|   Filter: \(\(title\)::text \~\~\* '%Core%'::text\) |
|   Rows Removed by Filter: 149999 |
| Planning Time: 0.298 ms |
| Execution Time: 68.788 ms |

    
*Объясните результат:*

Последовательное сканирование всех строк таблицы заняло значительное время.

17. Попробуйте удалить все индексы:
    ```sql
    DO $$ 
    DECLARE
        r RECORD;
    BEGIN
        FOR r IN (SELECT indexname FROM pg_indexes 
                  WHERE tablename = 't_books' 
                  AND indexname != 'books_pkey')
        LOOP
            EXECUTE 'DROP INDEX ' || r.indexname;
        END LOOP;
    END $$;
    ```
    
    *Результат:*
    ```
    workshop.public> DO $$
                     DECLARE
                         r RECORD;
                     BEGIN
                         FOR r IN (SELECT indexname FROM pg_indexes
                                   WHERE tablename = 't_books'
                                     AND indexname != 't_books_id_pk')
                             LOOP
                                 EXECUTE 'DROP INDEX ' || r.indexname;
                             END LOOP;
                     END $$ LANGUAGE plpgsql
    [2024-12-08 01:03:06] completed in 16 ms
    ```
*Объясните результат:*
Я модифицировал запрос, чтобы он удалял все индексы, кроме первичного ключа, так как на него наложено ограничение.


18. Создайте индекс для оптимизации суффиксного поиска:
    ```sql
    -- Вариант 1: с reverse()
    CREATE INDEX t_books_rev_title_idx ON t_books(reverse(title));
    
    -- Вариант 2: с триграммами
    CREATE EXTENSION IF NOT EXISTS pg_trgm;
    CREATE INDEX t_books_trgm_idx ON t_books USING gin (title gin_trgm_ops);
    ```
    
    *Результаты тестов:*

1

| QUERY PLAN |
| :--- |
| Seq Scan on t\_books  \(cost=0.00..3475.00 rows=750 width=33\) \(actual time=25.466..25.468 rows=1 loops=1\) |
|   Filter: \(reverse\(\(title\)::text\) \~\~ 'eroC%'::text\) |
|   Rows Removed by Filter: 149999 |
| Planning Time: 0.484 ms |
| Execution Time: 25.484 ms |


2

| QUERY PLAN |
| :--- |
| Bitmap Heap Scan on t\_books  \(cost=21.57..76.78 rows=15 width=33\) \(actual time=0.099..0.100 rows=1 loops=1\) |
|   Recheck Cond: \(\(title\)::text \~\~\* '%Core%'::text\) |
|   Heap Blocks: exact=1 |
|   -&gt;  Bitmap Index Scan on t\_books\_trgm\_idx  \(cost=0.00..21.56 rows=15 width=0\) \(actual time=0.087..0.087 rows=1 loops=1\) |
|         Index Cond: \(\(title\)::text \~\~\* '%Core%'::text\) |
| Planning Time: 0.316 ms |
| Execution Time: 0.124 ms |
    
*Объясните результаты:*
Второй вариант с триграммами существенно улучшает производительность запроса.


19. Выполните поиск по точному совпадению:
    ```sql
    EXPLAIN ANALYZE
    SELECT * FROM t_books WHERE title = 'Oracle Core';
    ```
    
    *План выполнения:*

| QUERY PLAN |
| :--- |
| Bitmap Heap Scan on t\_books  \(cost=116.57..120.58 rows=1 width=33\) \(actual time=0.052..0.053 rows=1 loops=1\) |
|   Recheck Cond: \(\(title\)::text = 'Oracle Core'::text\) |
|   Heap Blocks: exact=1 |
|   -&gt;  Bitmap Index Scan on t\_books\_trgm\_idx  \(cost=0.00..116.57 rows=1 width=0\) \(actual time=0.042..0.042 rows=1 loops=1\) |
|         Index Cond: \(\(title\)::text = 'Oracle Core'::text\) |
| Planning Time: 0.258 ms |
| Execution Time: 0.083 ms |


*Объясните результат:*
Индекс успешно улучшил производительность поиска.

20. Выполните поиск по началу названия:
    ```sql
    EXPLAIN ANALYZE
    SELECT * FROM t_books WHERE title ILIKE 'Relational%';
    ```
    
    *План выполнения:*

| QUERY PLAN |
| :--- |
| Bitmap Heap Scan on t\_books  \(cost=95.15..150.36 rows=15 width=33\) \(actual time=0.037..0.037 rows=0 loops=1\) |
|   Recheck Cond: \(\(title\)::text \~\~\* 'Relational%'::text\) |
|   Rows Removed by Index Recheck: 1 |
|   Heap Blocks: exact=1 |
|   -&gt;  Bitmap Index Scan on t\_books\_trgm\_idx  \(cost=0.00..95.15 rows=15 width=0\) \(actual time=0.023..0.023 rows=1 loops=1\) |
|         Index Cond: \(\(title\)::text \~\~\* 'Relational%'::text\) |
| Planning Time: 0.291 ms |
| Execution Time: 0.084 ms |

    
*Объясните результат:*
Индекс с триграммами успешно улучшил производительность поиска для запроса с ILIKE.

21. Создайте свой пример индекса с обратной сортировкой:
    ```sql
    CREATE INDEX t_books_desc_idx ON t_books(title DESC);
    ```
    
    *Тестовый запрос:*
    ```sql
    EXPLAIN ANALYZE
    SELECT * FROM t_books ORDER BY title DESC LIMIT 10;
    ```
    
    *План выполнения:*

| QUERY PLAN |
| :--- |
| Limit  \(cost=0.42..1.02 rows=10 width=33\) \(actual time=0.128..0.132 rows=10 loops=1\) |
|   -&gt;  Index Scan using t\_books\_desc\_idx on t\_books  \(cost=0.42..9074.60 rows=150000 width=33\) \(actual time=0.127..0.131 rows=10 loops=1\) |
| Planning Time: 0.352 ms |
| Execution Time: 0.145 ms |

    
*Объясните результат:*
Индекс с обратной сортировкой значительно улучшил производительность запроса.