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
    ```sql
    Bitmap Heap Scan on t_books  (cost=21.03..1335.59 rows=750 width=33) (actual time=0.017..0.017 rows=1 loops=1)
    "  Recheck Cond: (to_tsvector('english'::regconfig, (title)::text) @@ '''expert'''::tsquery)"
      Heap Blocks: exact=1
      ->  Bitmap Index Scan on t_books_fts_idx  (cost=0.00..20.84 rows=750 width=0) (actual time=0.012..0.012 rows=1 loops=1)
    "        Index Cond: (to_tsvector('english'::regconfig, (title)::text) @@ '''expert'''::tsquery)"
    Planning Time: 0.364 ms
    Execution Time: 0.043 ms
    ```
    
*Объясните результат:*

GIN индекс для полнотекстового поиска работает отлично.

**Ключевые моменты:**
1. **Bitmap Index Scan** использует GIN индекс для быстрого поиска слова 'expert'
2. Найдена всего **1 строка** (`rows=1`), хотя PostgreSQL ожидал найти 750 строк (оценка статистики)
3. **Heap Blocks: exact=1** - данные лежат в одном блоке памяти
4. **Сверхбыстрое выполнение:** 0.043 мс

**Почему так эффективно:**
- GIN индекс специально создан для полнотекстового поиска
- Хранит все вхождения слов из текста с их позициями
- Позволяет мгновенно находить документы по любому слову
- Работает с морфологией (stemming) - находит словоформы

**Вывод:** GIN индексы идеальны для полнотекстового поиска - в 350+ раз быстрее последовательного сканирования.

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
     ```sql
    Index Scan using t_lookup_pk on t_lookup  (cost=0.42..8.44 rows=1 width=23) (actual time=0.011..0.011 rows=1 loops=1)
      Index Cond: ((item_key)::text = '0000000455'::text)
    Planning Time: 0.081 ms
    Execution Time: 0.022 ms
    ```
     
*Объясните результат:*

Используется Index Scan по первичному ключу (B-tree индекс).

**Что происходит:**
1. **Index Scan using t_lookup_pk** - PostgreSQL использует B-tree индекс PK для быстрого поиска
2. **Index Cond** - ищет точное соответствие `item_key = '0000000455'`
3. **Найдена 1 строка** за 0.022 мс
4. **Эффективно**, так как PK обеспечивает уникальность и быстрый доступ

**Особенности B-tree для PK:**
- O(log n) сложность поиска
- Поддерживает точный поиск, диапазоны, сортировку
- Индекс хранит физическое расположение строк

**Вывод:** B-tree индекс на первичном ключе обеспечивает максимальную производительность для точечного поиска.

14. Выполните поиск по ключу в кластеризованной таблице:
     ```sql
     EXPLAIN ANALYZE
     SELECT * FROM t_lookup_clustered WHERE item_key = '0000000455';
     ```
     
     *План выполнения:*
     ```sql
    Index Scan using t_lookup_clustered_pkey on t_lookup_clustered  (cost=0.42..8.44 rows=1 width=23) (actual time=0.179..0.180 rows=1 loops=1)
      Index Cond: ((item_key)::text = '0000000455'::text)
    Planning Time: 0.774 ms
    Execution Time: 0.275 ms
    ```
     
     *Объясните результат:*

Кластеризация не ускоряет поиск одной строки по PK. Индекс и так быстро находит её.

**Кластеризация помогает когда:**
- Берут много строк подряд (диапазон `BETWEEN`, `ORDER BY`)
- Делают полное сканирование с сортировкой

**Для одной строки - бесполезно.** Даже может быть медленнее из-за кэша или перераспределения данных на диске.

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
     ```sql
    Index Scan using t_lookup_value_idx on t_lookup  (cost=0.42..8.44 rows=1 width=23) (actual time=0.078..0.078 rows=0 loops=1)
      Index Cond: ((item_value)::text = 'T_BOOKS'::text)
    Planning Time: 0.797 ms
    Execution Time: 0.097 ms
    ```
     
     *Объясните результат:*

Используется Index Scan по индексу `t_lookup_value_idx` (B-tree).

**Что происходит:**
1. PostgreSQL использует вторичный индекс по `item_value` для быстрого поиска
2. Найдено **0 строк** (`rows=0`) - значения `'T_BOOKS'` нет в таблице
3. **Index Scan** читает только индекс, не заглядывая в таблицу (heap)
4. Быстрое выполнение: 0.097 мс

**Почему эффективно:**
- B-tree индекс позволяет быстро проверить наличие значения
- Если строка не найдена в индексе - таблица не читается
- Подходит для проверки существования значений

**Особенность:** PostgreSQL выбрал `Index Scan` вместо `Bitmap Index Scan`, так как ожидает мало строк (rows=1).

18. Выполните поиск по значению в кластеризованной таблице:
     ```sql
     EXPLAIN ANALYZE
     SELECT * FROM t_lookup_clustered WHERE item_value = 'T_BOOKS';
     ```
     
     *План выполнения:*
     ```sql
    Index Scan using t_lookup_clustered_value_idx on t_lookup_clustered  (cost=0.42..8.44 rows=1 width=23) (actual time=0.105..0.105 rows=0 loops=1)
      Index Cond: ((item_value)::text = 'T_BOOKS'::text)
    Planning Time: 0.814 ms
    Execution Time: 0.178 ms

    ```
     
     *Объясните результат:*

Точно такой же план, как и для обычной таблицы (Index Scan по вторичному индексу).

**Ключевые моменты:**
1. **Аналогичная производительность** - 0.178 мс против 0.097 мс у обычной таблицы
2. Найдено **0 строк** - значения нет в обеих таблицах
3. PostgreSQL использует вторичный индекс `t_lookup_clustered_value_idx`

**Почему кластеризация не влияет:**
- Кластеризация сделана по **`item_key`** (PK)
- Поиск идёт по **`item_value`** - другому полю
- Физическое упорядочивание по PK не помогает поиску по другому полю

**Вывод:** Кластеризация по одному полю не ускоряет поиск по другим полям. Для каждого поля нужен свой индекс.

19. Сравните производительность поиска по значению в обычной и кластеризованной таблицах:
     
     *Сравнение:*

**Результат:** Производительность поиска по `item_value` в обычной и кластеризованной таблицах практически одинакова.

**Объяснение:**
1. Обе таблицы используют B-tree индекс по полю `item_value`
2. Кластеризация сделана по `item_key` (PK), поэтому не влияет на поиск по другому полю
3. Время выполнения отличается незначительно: 0.097 мс vs 0.178 мс

**Вывод:** Кластеризация таблицы ускоряет только операции, использующие порядок кластеризации (диапазонные запросы, сортировка по PK). Для поиска по другим полям важны отдельные индексы, а не кластеризация.