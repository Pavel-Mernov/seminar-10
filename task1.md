# Задание 1. B-tree индексы в PostgreSQL

1. Запустите БД через docker compose в ./src/docker-compose.yml:

2. Выполните запрос для поиска книги с названием 'Oracle Core' и получите план выполнения:
   ```sql
   EXPLAIN ANALYZE
   SELECT * FROM t_books WHERE title = 'Oracle Core';
   ```
   
   *План выполнения:*
   ![image](https://github.com/user-attachments/assets/b660b5c3-2438-4dfc-8c18-5395f6ec9ee5)

   
   *Объясните результат:*
   1. Проход по всем записям таблицы (без индексов).
   2. Записи фильтруется согласно условию (название книги).

   Операция поиска строк проходит слишком долго ($21$ секунд), что значительно превышает ожидаемое время ($1.184$ сек).

   Максимальный **cost** равен $3100.00$ **(очень много)**.

   **Итого: Без индексов работает медленно и неэффективно.**

4. Создайте B-tree индексы:
   ```sql
   CREATE INDEX t_books_title_idx ON t_books(title);
   CREATE INDEX t_books_active_idx ON t_books(is_active);
   ```
   
   *Результат:*
   ![image](https://github.com/user-attachments/assets/4619457d-8e5a-4cb0-a1a6-45322e46667e)


5. Проверьте информацию о созданных индексах:
   ```sql
   SELECT schemaname, tablename, indexname, indexdef
   FROM pg_catalog.pg_indexes
   WHERE tablename = 't_books';
   ```
   
   *Результат:*
   ![image](https://github.com/user-attachments/assets/aa91c554-578f-47c2-8969-1c5cfb7d9762)
   
   *Объясните результат:*
   Создан индекс $t_books_id_pk$ (тип $integer$) на поле $books_id$, требующий **уникальности** значений.
   Создан индекс на поле $title$ (тип $varchar$).
   Создан индекс на поле $is_active$ (тип $boolean$).

7. Обновите статистику таблицы:
   ```sql
   ANALYZE t_books;
   ```
   
   *Результат:*
   ![image](https://github.com/user-attachments/assets/9c00e049-ad4a-4888-b850-801d6bfd2113)

8. Выполните запрос для поиска книги 'Oracle Core' и получите план выполнения:
   ```sql
   EXPLAIN ANALYZE
   SELECT * FROM t_books WHERE title = 'Oracle Core';
   ```
   
   *План выполнения:*
   ![image](https://github.com/user-attachments/assets/31bbf9f4-8518-41ba-bd42-519e629f578b)
   
   *Объясните результат:*
   Фактическое время выполнения: меньше секунды (ожидаемое время - около $1.5$ секунд.).
   Максимальный $cost$ - уже $8.44$ - **лучше** чем без индексов.

   **Вывод:**

   Индекс работает **эффективно** при применении оператора $=$ для сравнения строк.

10. Выполните запрос для поиска книги по book_id и получите план выполнения:
   ```sql
   EXPLAIN ANALYZE
   SELECT * FROM t_books WHERE book_id = 18;
   ```
   
   *План выполнения:*
   ![image](https://github.com/user-attachments/assets/8174bd00-7fd3-4818-af9b-896114fdbfcf)

   *Объясните результат:*
   Фактическое время выполнения: $более 7$ секунд (ожидаемое время - меньше $1$ секунды.) (**неэффективно по времени**).
   Максимальный $cost$ - уже $8.44$ - **так же, как при поиске по title**.

   **Вывод:**

   Индекс работает **эффективно по затратам на операциии**, но **неэффективно по времени** при поиске по $id$ (т.к. $id$ - **первичный ключ** и, следовательно, уникальное поле, т.е. никакие 2 книги не имеют одинаковые $id$).

11. Выполните запрос для поиска активных книг и получите план выполнения:
   ```sql
   EXPLAIN ANALYZE
   SELECT * FROM t_books WHERE is_active = true;
   ```
   
   *План выполнения:*
   ![image](https://github.com/user-attachments/assets/79dc91bb-bb1f-495f-8dba-ddc72c500411)

   
   *Объясните результат:*
   Время выполнения - $70.066$ секунд (ожидается $1.4$ секунды) - **очень долго**.
   Затраты ($cost$) - $2725.00$ - **тоже очень много**.

   **Вывод**

   Индекс работает **неэффективно**, т.к. тип $boolean$ принимает **всего 2 значения** - $True$ и $False$.
 
11. Посчитайте количество строк и уникальных значений:
   ```sql
   SELECT 
       COUNT(*) as total_rows,
       COUNT(DISTINCT title) as unique_titles,
       COUNT(DISTINCT category) as unique_categories,
       COUNT(DISTINCT author) as unique_authors
   FROM t_books;
   ```
   
   *Результат:*
   ![image](https://github.com/user-attachments/assets/1b310ecf-a545-4245-93ee-80435bfacf46)


11. Удалите созданные индексы:
    ```sql
    DROP INDEX t_books_title_idx;
    DROP INDEX t_books_active_idx;
    ```
    
    *Результат:*
    ![image](https://github.com/user-attachments/assets/20fe6db4-b852-4cb3-b686-204018fd0af3)

12. Основываясь на предыдущих результатах, создайте индексы для оптимизации следующих запросов:
    a. `WHERE title = $1 AND category = $2`
    b. `WHERE title = $1`
    c. `WHERE category = $1 AND author = $2`
    d. `WHERE author = $1 AND book_id = $2`
    
    *Созданные индексы:*
    [Вставьте команды создания индексов]
    
    *Объясните ваше решение:*
    [Ваше объяснение]

13. Протестируйте созданные индексы.
    
    *Результаты тестов:*
    [Вставьте планы выполнения для каждого случая]
    
    *Объясните результаты:*
    [Ваше объяснение]

14. Выполните регистронезависимый поиск по началу названия:
    ```sql
    EXPLAIN ANALYZE
    SELECT * FROM t_books WHERE title ILIKE 'Relational%';
    ```
    
    *План выполнения:*
    [Вставьте план выполнения]
    
    *Объясните результат:*
    [Ваше объяснение]

15. Создайте функциональный индекс:
    ```sql
    CREATE INDEX t_books_up_title_idx ON t_books(UPPER(title));
    ```
    
    *Результат:*
    [Вставьте результат выполнения]

16. Выполните запрос из шага 13 с использованием UPPER:
    ```sql
    EXPLAIN ANALYZE
    SELECT * FROM t_books WHERE UPPER(title) LIKE 'RELATIONAL%';
    ```
    
    *План выполнения:*
    [Вставьте план выполнения]
    
    *Объясните результат:*
    [Ваше объяснение]

17. Выполните поиск подстроки:
    ```sql
    EXPLAIN ANALYZE
    SELECT * FROM t_books WHERE title ILIKE '%Core%';
    ```
    
    *План выполнения:*
    [Вставьте план выполнения]
    
    *Объясните результат:*
    [Ваше объяснение]

18. Попробуйте удалить все индексы:
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
    [Вставьте результат выполнения]
    
    *Объясните результат:*
    [Ваше объяснение]

19. Создайте индекс для оптимизации суффиксного поиска:
    ```sql
    -- Вариант 1: с reverse()
    CREATE INDEX t_books_rev_title_idx ON t_books(reverse(title));
    
    -- Вариант 2: с триграммами
    CREATE EXTENSION IF NOT EXISTS pg_trgm;
    CREATE INDEX t_books_trgm_idx ON t_books USING gin (title gin_trgm_ops);
    ```
    
    *Результаты тестов:*
    [Вставьте планы выполнения для обоих вариантов]
    
    *Объясните результаты:*
    [Ваше объяснение]

20. Выполните поиск по точному совпадению:
    ```sql
    EXPLAIN ANALYZE
    SELECT * FROM t_books WHERE title = 'Oracle Core';
    ```
    
    *План выполнения:*
    [Вставьте план выполнения]
    
    *Объясните результат:*
    [Ваше объяснение]

21. Выполните поиск по началу названия:
    ```sql
    EXPLAIN ANALYZE
    SELECT * FROM t_books WHERE title ILIKE 'Relational%';
    ```
    
    *План выполнения:*
    [Вставьте план выполнения]
    
    *Объясните результат:*
    [Ваше объяснение]

22. Создайте свой пример индекса с обратной сортировкой:
    ```sql
    CREATE INDEX t_books_desc_idx ON t_books(title DESC);
    ```
    
    *Тестовый запрос:*
    [Вставьте ваш запрос]
    
    *План выполнения:*
    [Вставьте план выполнения]
    
    *Объясните результат:*
    [Ваше объяснение]
