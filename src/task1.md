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
   ```sql
    Bitmap Heap Scan on t_books  (cost=12.00..16.01 rows=1 width=33) (actual time=0.010..0.010 rows=0 loops=1)
      Recheck Cond: (category IS NULL)
      ->  Bitmap Index Scan on t_books_brin_cat_idx  (cost=0.00..12.00 rows=1 width=0) (actual time=0.008..0.008 rows=0 loops=1)
            Index Cond: (category IS NULL)
    Planning Time: 0.219 ms
    Execution Time: 0.023 ms
    ```
   
   *Объясните результат:*

BRIN индекс эффективно находит NULL-значения, так как они хранятся в метаданных диапазонов. 

План показывает:

Bitmap Index Scan использует BRIN индекс для быстрого определения диапазонов страниц, которые могут содержать NULL

Bitmap Heap Scan затем проверяет найденные страницы и фильтрует точные строки

Потери производительности минимальны, поскольку BRIN быстро отбрасывает диапазоны без NULL

6. Создайте BRIN индекс по автору:
   ```sql
   CREATE INDEX t_books_brin_author_idx ON t_books USING brin(author);
   ```

7. Выполните поиск по категории и автору:
   ```sql
   EXPLAIN ANALYZE
   SELECT * FROM t_books 
   WHERE category = 'INDEX' AND author = 'SYSTEM';
   ```
   
   *План выполнения:*
   ```sql
    Bitmap Heap Scan on t_books  (cost=12.17..2400.34 rows=1 width=33) (actual time=15.667..15.668 rows=0 loops=1)
      Recheck Cond: ((category)::text = 'INDEX'::text)
      Rows Removed by Index Recheck: 150000
      Filter: ((author)::text = 'SYSTEM'::text)
      Heap Blocks: lossy=1225
      ->  Bitmap Index Scan on t_books_brin_cat_idx  (cost=0.00..12.17 rows=77545 width=0) (actual time=0.087..0.087 rows=12250 loops=1)
            Index Cond: ((category)::text = 'INDEX'::text)
    Planning Time: 0.258 ms
    Execution Time: 15.686 ms
    ```
   
   *Объясните результат (обратите внимание на bitmap scan):*

**Проблема:** BRIN индекс по `category` вернул ~1225 блоков (страниц) данных, в которых PostgreSQL проверил все 150,000 строк, но ни одна не соответствовала условию `author = 'SYSTEM'`.

**Почему так произошло:**
1. BRIN работает с блоками (по 8КБ), а не отдельными строками
2. Если данные в таблице НЕ отсортированы по `category`, то в каждом блоке будут разные категории
3. Для условия `author = 'SYSTEM'` используется фильтрация в памяти, а не индексный поиск

**Вывод:** BRIN эффективен только при сильной корреляции данных в таблице с индексируемым полем. В данном случае данные плохо кластеризованы по `category`, поэтому BRIN почти бесполезен.

8. Получите список уникальных категорий:
   ```sql
   EXPLAIN ANALYZE
   SELECT DISTINCT category 
   FROM t_books 
   ORDER BY category;
   ```
   
   *План выполнения:*
    ```sql
    Sort  (cost=3100.14..3100.15 rows=6 width=7) (actual time=29.753..29.754 rows=6 loops=1)
      Sort Key: category
      Sort Method: quicksort  Memory: 25kB
      ->  HashAggregate  (cost=3100.00..3100.06 rows=6 width=7) (actual time=29.684..29.686 rows=6 loops=1)
            Group Key: category
            Batches: 1  Memory Usage: 24kB
            ->  Seq Scan on t_books  (cost=0.00..2725.00 rows=150000 width=7) (actual time=0.012..9.788 rows=150000 loops=1)
    Planning Time: 0.095 ms
    Execution Time: 29.835 ms
    ```
   
   *Объясните результат:*

PostgreSQL игнорирует BRIN индекс и делает полное сканирование таблицы (Seq Scan).

**Причины:**
1. Для `DISTINCT` + `ORDER BY` нужно прочитать ВСЕ значения категорий
2. BRIN не может вернуть отсортированные уникальные значения - он работает только с фильтрацией
3. Если бы данные были идеально отсортированы по `category`, BRIN мог бы помочь пропустить некоторые блоки, но даже тогда выгода минимальна
4. PostgreSQL считает, что прочитать всю таблицу (150к строк) быстрее, чем использовать BRIN с последующей дорогой повторной проверкой строк

**Вывод:** BRIN бесполезен для операций `DISTINCT`/`GROUP BY`/`ORDER BY` на полном наборе данных.

9. Подсчитайте книги, где автор начинается на 'S':
   ```sql
   EXPLAIN ANALYZE
   SELECT COUNT(*) 
   FROM t_books 
   WHERE author LIKE 'S%';
   ```
   
   *План выполнения:*
   ```sql
    Aggregate  (cost=3100.04..3100.05 rows=1 width=8) (actual time=15.677..15.678 rows=1 loops=1)
      ->  Seq Scan on t_books  (cost=0.00..3100.00 rows=15 width=0) (actual time=15.673..15.673 rows=0 loops=1)
            Filter: ((author)::text ~~ 'S%'::text)
            Rows Removed by Filter: 150000
    Planning Time: 0.631 ms
    Execution Time: 15.871 ms
    ```
   
   *Объясните результат:*

PostgreSQL снова игнорирует BRIN индекс по `author` и делает полное сканирование таблицы.

**Причины:**
1. BRIN индекс **не поддерживает префиксный поиск** (`LIKE 'S%'`)
2. BRIN работает только с операторами равенства (`=`) и диапазонов (`<`, `>`, `BETWEEN`)
3. Для префиксного поиска нужен B-tree индекс
4. PostgreSQL читает все 150к строк и фильтрует их в памяти

**Вывод:** BRIN бесполезен для LIKE-запросов с паттернами в начале строки. Только B-tree/GIN индексы поддерживают такой тип поиска.

10. Создайте индекс для регистронезависимого поиска:
    ```sql
    CREATE INDEX t_books_lower_title_idx ON t_books(LOWER(title));
    ```

11. Подсчитайте книги, начинающиеся на 'O':
    ```sql
    EXPLAIN ANALYZE
    SELECT COUNT(*) 
    FROM t_books 
    WHERE LOWER(title) LIKE 'o%';
    ```
   
   *План выполнения:*
   ```sql
    Aggregate  (cost=3476.88..3476.89 rows=1 width=8) (actual time=28.296..28.297 rows=1 loops=1)
      ->  Seq Scan on t_books  (cost=0.00..3475.00 rows=750 width=0) (actual time=28.291..28.292 rows=1 loops=1)
            Filter: (lower((title)::text) ~~ 'o%'::text)
            Rows Removed by Filter: 149999
    Planning Time: 0.062 ms
    Execution Time: 28.315 ms
    
```
   
   *Объясните результат:*

Созданный индекс `t_books_lower_title_idx` НЕ используется.

**Причины:**
1. **Несоответствие типа данных:** В плане видно `lower((title)::text)` - PostgreSQL делает явное приведение типа `::text`, что может мешать использованию индекса
2. **Статистика/селективность:** Если PostgreSQL считает, что условие `LIKE 'o%'` вернет много строк (оценка 750), он может выбрать полное сканирование

**Вывод:** Даже с созданным функциональным индексом PostgreSQL может выбрать полное сканирование, если считает его более эффективным для данного запроса.

12. Удалите созданные индексы:
    ```sql
    DROP INDEX t_books_brin_cat_idx;
    DROP INDEX t_books_brin_author_idx;
    DROP INDEX t_books_lower_title_idx;
    ```

13. Создайте составной BRIN индекс:
    ```sql
    CREATE INDEX t_books_brin_cat_auth_idx ON t_books 
    USING brin(category, author);
    ```

14. Повторите запрос из шага 7:
    ```sql
    EXPLAIN ANALYZE
    SELECT * FROM t_books 
    WHERE category = 'INDEX' AND author = 'SYSTEM';
    ```
   
   *План выполнения:*
   ```sql
    Bitmap Heap Scan on t_books  (cost=12.17..2400.34 rows=1 width=33) (actual time=1.714..1.715 rows=0 loops=1)
      Recheck Cond: (((category)::text = 'INDEX'::text) AND ((author)::text = 'SYSTEM'::text))
      Rows Removed by Index Recheck: 8834
      Heap Blocks: lossy=73
      ->  Bitmap Index Scan on t_books_brin_cat_auth_idx  (cost=0.00..12.17 rows=77545 width=0) (actual time=0.021..0.022 rows=730 loops=1)
            Index Cond: (((category)::text = 'INDEX'::text) AND ((author)::text = 'SYSTEM'::text))
    Planning Time: 0.203 ms
    Execution Time: 1.754 ms
```
    
*Объясните результат:*

Составной BRIN индекс работает ЛУЧШЕ, чем два отдельных (шаг 7).

**Улучшения:**
1. **Время выполнения:** 1.75 мс против 15.7 мс (в ~9 раз быстрее)
2. **Блоков проверено:** 73 против 1225 (в ~17 раз меньше)
3. **Строк перепроверено:** 8,834 против 150,000 (в ~17 раз меньше)

**Почему так:**
1. Составной индекс учитывает КОМБИНАЦИЮ `(category, author)`
2. BRIN точнее определяет блоки, где одновременно `category='INDEX'` И `author='SYSTEM'`
3. Меньше "ложных срабатываний" - блоков, где только одно условие истинно

**Вывод:** Составные BRIN индексы могут быть эффективны для многокритериальных запросов, если данные имеют корреляцию по этим полям.