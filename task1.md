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
   Фактическое время выполнения: меньше 1 миллисекунды (ожидаемое время - около $1.5$ мсек.).
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
   Фактическое время выполнения: $более 7$ мсек (ожидаемое время - меньше $1$ мсек.) (**неэффективно по времени**).
   Максимальный $cost$ - уже $8.44$ - **так же, как при поиске по title**.

   **Вывод:**

   Индекс работает **эффективно по затратам на операциии**, но **неэффективно по времени** при поиске по $id$ (т.к. $id$ - **первичный ключ** и, следовательно, уникальное поле, т.е. никакие 2 книги не имеют одинаковые $id$; другое возможное объяснение - это тип поля $id$ - **Integer**, поэтому неэффективно).

11. Выполните запрос для поиска активных книг и получите план выполнения:
   ```sql
   EXPLAIN ANALYZE
   SELECT * FROM t_books WHERE is_active = true;
   ```
   
   *План выполнения:*
   ![image](https://github.com/user-attachments/assets/79dc91bb-bb1f-495f-8dba-ddc72c500411)

   
   *Объясните результат:*
   Время выполнения - $70.066$ секунд (ожидается $1.4$ мсек) - **очень долго**.
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
    ![image](https://github.com/user-attachments/assets/df2ceabb-6de7-4acf-ba00-d9a1ce66fd42)
    ![image](https://github.com/user-attachments/assets/edd8f8ff-f4d1-4c0d-89d7-6fd3c1a13f14)
    ![image](https://github.com/user-attachments/assets/d775eb67-4051-4ed4-80d7-127e3b37ca8d)

    
    *Объясните ваше решение:*
    Создал индексы на поля **title**, **author** и **category**. Все эти 3 поля имеют тип **varchar**, а при проверке на равенство **varchar** индексы показали наибольшую эффективность.

    Индексы на **id** добавлять не стал, т.к. тут есть требование на уникальность, да еще тип **integer**. Предущие опыты показали низкую эффективность индексов в таких условиях.

14. Протестируйте созданные индексы.
    
    *Результаты тестов:*
    ![image](https://github.com/user-attachments/assets/1cd34f4a-57c5-4de1-86cf-375176d4d042)
    ![image](https://github.com/user-attachments/assets/b579d814-1440-4d0b-98f0-345355d03819)
    ![image](https://github.com/user-attachments/assets/3acbef6e-53f9-44f5-aa63-e39cb4bb45f6)
    ![image](https://github.com/user-attachments/assets/7c5de449-af6b-412e-9274-98fd8827d25f)

    
    *Объясните результаты:*
    На все запросы, кроме `WHERE category = $1 AND author = $2`, СУБД отвечала быстро (меньше 1 мсек) - так и предполагалось.

    На запрос `WHERE category = $1 AND author = $2` был довольно долгий ответ ($1.6$ мсек), ввиду конфликта индексов на тип **varchar**.

16. Выполните регистронезависимый поиск по началу названия:
    ```sql
    EXPLAIN ANALYZE
    SELECT * FROM t_books WHERE title ILIKE 'Relational%';
    ```
    
    *План выполнения:*
    ![image](https://github.com/user-attachments/assets/1d2e1c20-b0f7-41dd-9fa1-55b381b299b3)

    
    *Объясните результат:*
    Поиск выполняется **очень долго** - $255$ миллисекунд.

    **Вывод**

    Стандартный индекс на тип **Varchar** плохо оптимизирует запросы на регистронезависимый поиск с оператором $ILIKE$.

18. Создайте функциональный индекс:
    ```sql
    CREATE INDEX t_books_up_title_idx ON t_books(UPPER(title));
    ```
    
    *Результат:*
    ![image](https://github.com/user-attachments/assets/d81e07a2-c814-427d-a43e-08192033213e)

19. Выполните запрос из шага 13 с использованием UPPER:
    ```sql
    EXPLAIN ANALYZE
    SELECT * FROM t_books WHERE UPPER(title) LIKE 'RELATIONAL%';
    ```
    
    *План выполнения:*
    ![image](https://github.com/user-attachments/assets/d2ef288b-42c0-4851-92d5-dd8eac1fb975)

    
    *Объясните результат:*
    Слишком долгий фактический поиск ($99$ мс), по сравнению с ожидаемым ($< 1$ мс). Возможно это связано с тем, что индекс создавался для функции $UPPER$, а при поиске используется более сложный алгоритм сравнения.

20. Выполните поиск подстроки:
    ```sql
    EXPLAIN ANALYZE
    SELECT * FROM t_books WHERE title ILIKE '%Core%';
    ```
    
    *План выполнения:*
    ![image](https://github.com/user-attachments/assets/fe46bdda-106f-499c-a882-58485824f900)
    
    *Объясните результат:*

    Опять же, значительно более долгий фактический поиск, по сравнению с ожидаемым, возможно, связан с тем, что индекс создавался для функции $UPPER$, а при поиске используется более сложный алгоритм сравнения.

21. Попробуйте удалить все индексы:
    ```sql
    DO $$ 
    DECLARE
        r RECORD;
    BEGIN
        FOR r IN (SELECT indexname FROM pg_indexes 
                  WHERE tablename = 't_books' 
                  AND indexname != 't_books_id_pk')
        LOOP
            EXECUTE 'DROP INDEX ' || r.indexname;
        END LOOP;
    END $$;
    ```
    
    *Результат:*
    ![image](https://github.com/user-attachments/assets/ee5d54f3-07b7-4e6d-b726-41b76f9f2b32)

    
    *Объясните результат:*
    Пришлось изменить запрос (имя индекса для $PRIMARY KEY$), т.к. попытка удаления индекса первичного ключа приводит к ошибке.

22. Создайте индекс для оптимизации суффиксного поиска:
    ```sql
    -- Вариант 1: с reverse()
    CREATE INDEX t_books_rev_title_idx ON t_books(reverse(title));
    
    -- Вариант 2: с триграммами
    CREATE EXTENSION IF NOT EXISTS pg_trgm;
    CREATE INDEX t_books_trgm_idx ON t_books USING gin (title gin_trgm_ops);
    ```
    
    *Результаты тестов:*
    ![image](https://github.com/user-attachments/assets/ba064928-8aba-40ae-9728-82ca87b9ad6d)

    ![image](https://github.com/user-attachments/assets/a9349b91-c8bd-42a7-bef6-143fcab0aa7a)

    
    *Объясните результаты:*
    Индексы на суффиксный поиск созданы успешно.

23. Выполните поиск по точному совпадению:
    ```sql
    EXPLAIN ANALYZE
    SELECT * FROM t_books WHERE title = 'Oracle Core';
    ```
    
    *План выполнения:*
    ![image](https://github.com/user-attachments/assets/abb10729-3e3b-4374-a044-32a118776795)

    
    *Объясните результат:*

    Фактическое время выполнения - $< 1$ мсек. (гораздо меньше, чем ожидаемое $> 2$ мсек). Проверка на равенство строк зарекомендовала себя как эффективный инструмент (в сочетании с индексом).

24. Выполните поиск по началу названия:
    ```sql
    EXPLAIN ANALYZE
    SELECT * FROM t_books WHERE title ILIKE 'Relational%';
    ```
    
    *План выполнения:*
    ![image](https://github.com/user-attachments/assets/9c856500-b967-482a-a58b-01cce1020a0c)

    
    *Объясните результат:*
    [Ваше объяснение]

25. Создайте свой пример индекса с обратной сортировкой:
    ```sql
    CREATE INDEX t_books_desc_idx ON t_books(title DESC);
    ```
    
    *Тестовый запрос:*
    [Вставьте ваш запрос]
    
    *План выполнения:*
    [Вставьте план выполнения]
    
    *Объясните результат:*
    [Ваше объяснение]
