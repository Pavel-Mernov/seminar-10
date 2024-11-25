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
   
   ![image](https://github.com/user-attachments/assets/46c0a49b-5b85-4d6b-b100-67a9a403701f)


5. Выполните запрос для поиска книги с id = 18:
   ```sql
   EXPLAIN ANALYZE
   SELECT * FROM t_books_part WHERE book_id = 18;
   ```
   
   *План выполнения:*
   
   ![image](https://github.com/user-attachments/assets/963140b8-d7ce-4383-b2f8-670ac5a8952c)
   
   *Объясните результат:*
   
   С использованием партиций (разбиения по значению $id$), фильтр по значению $id$ происходит **лучше**, чем без партиций и без индексов (за счёт разделения по значению), но **хуже**, чем с использованием индексов (как по $cost$, так и по $времени \ выполнения$).

6. Выполните поиск по названию книги:
   ```sql
   EXPLAIN ANALYZE
   SELECT * FROM t_books_part 
   WHERE title = 'Expert PostgreSQL Architecture';
   ```
   
   *План выполнения:*
   
   ![image](https://github.com/user-attachments/assets/1c7df428-a10a-48ef-a1a0-4bf72e2949b5)

   
   *Объясните результат:*
   
   Поиск по названию происходит **неэффективно по времени**, т.к. разделение было по *индексу*, а не по *названию*.

   При поиске, произошёл перебор **Всех** сегментов по *индексу*, поэтому так неэффективно.

8. Создайте партиционированный индекс:
   ```sql
   CREATE INDEX ON t_books_part(title);
   ```
   
   *Результат:*
   
   ![image](https://github.com/user-attachments/assets/e029216a-3867-4962-82bd-268444adec0b)


10. Повторите запрос из шага 5:
   ```sql
   EXPLAIN ANALYZE
   SELECT * FROM t_books_part 
   WHERE title = 'Expert PostgreSQL Architecture';
   ```
   
   *План выполнения:*
   
   ![image](https://github.com/user-attachments/assets/f95213e4-1da2-4968-813c-434fe4243f53)

   
   *Объясните результат:*
   
   С индексом на название книги даже проход по всем трём *partition* проходит очень быстро - всего за $0.094$ мсек.

11. Удалите созданный индекс:
   ```sql
   DROP INDEX t_books_part_title_idx;
   ```
   
   *Результат:*
   
   ![image](https://github.com/user-attachments/assets/79efa95a-301e-43dd-8953-4a758f3cc6d0)


11. Создайте индекс для каждой партиции:
   ```sql
   CREATE INDEX ON t_books_part_1(title);
   CREATE INDEX ON t_books_part_2(title);
   CREATE INDEX ON t_books_part_3(title);
   ```
   
   *Результат:*
   
   ![image](https://github.com/user-attachments/assets/506fc23d-fd97-4bd7-a1c1-46d5c7889558)


11. Повторите запрос из шага 5:
    ```sql
    EXPLAIN ANALYZE
    SELECT * FROM t_books_part 
    WHERE title = 'Expert PostgreSQL Architecture';
    ```
    
    *План выполнения:*
    
    ![image](https://github.com/user-attachments/assets/05580df1-72ab-4487-a27e-ae8385e03e18)

    
    *Объясните результат:*
    
    Просканирован по индексу каждый из 3-х сегментов *partition'а*. Однако затраты на сканирование каждого patrition'а довольно низкие (итого за все сегменты $0.124$ мсек), что несколько **выше,** чем при индексе без учёта *partition*-ов.

13. Удалите созданные индексы:
    ```sql
    DROP INDEX t_books_part_1_title_idx;
    DROP INDEX t_books_part_2_title_idx;
    DROP INDEX t_books_part_3_title_idx;
    ```
    
    *Результат:*
    
    ![image](https://github.com/user-attachments/assets/75df5099-6320-4de4-b9a9-2e8883eca84e)


14. Создайте обычный индекс по book_id:
    ```sql
    CREATE INDEX t_books_part_idx ON t_books_part(book_id);
    ```
    
    *Результат:*
    
    ![image](https://github.com/user-attachments/assets/7ba0ceeb-2f31-42b6-a54f-4a3596b93520)

15. Выполните поиск по book_id:
    ```sql
    EXPLAIN ANALYZE
    SELECT * FROM t_books_part WHERE book_id = 11011;
    ```
    
    *План выполнения:*
    
    ![image](https://github.com/user-attachments/assets/a4126de7-e9da-4a6c-87ac-3f0156a32c9f)
    
    *Объясните результат:*
    
    Сканирование по индексу только одного из сегментов *partition* ($t_books_part_1$), т.к. в фильтре поиска указан $id$.

    Фактическое время выполнения -  около $0.1$ мсек, при ожидаемом времени - $1.3$ мсек. **Очень эффективно по времени**

16. Создайте индекс по полю is_active:
    ```sql
    CREATE INDEX t_books_active_idx ON t_books(is_active);
    ```
    
    *Результат:*
    
    ![image](https://github.com/user-attachments/assets/377a3cc1-3c5a-41bf-8e58-61620c76eb3b)


18. Выполните поиск активных книг с отключенным последовательным сканированием:
    ```sql
    SET enable_seqscan = off;
    EXPLAIN ANALYZE
    SELECT * FROM t_books WHERE is_active = true;
    SET enable_seqscan = on;
    ```
    
    *План выполнения:*
    
    ![image](https://github.com/user-attachments/assets/02fcf4a5-6dee-42f0-abd1-73b8e952238e)

    
    *Объясните результат:*
    
    Произошёл Bitmap Heap Scan, в сочетании с индексом на тип **Boolean**.

    Внутри Heap Scan произошёл Bitmap Index Scan. 

    Итого время выполнения - $78$ мсек - **очень долго**.

20. Создайте составной индекс:
    ```sql
    CREATE INDEX t_books_author_title_index ON t_books(author, title);
    ```
    
    *Результат:*
    
    ![image](https://github.com/user-attachments/assets/20596923-57f5-435d-9c37-ef44b4b4ad17)

21. Найдите максимальное название для каждого автора:
    ```sql
    EXPLAIN ANALYZE
    SELECT author, MAX(title) 
    FROM t_books 
    GROUP BY author;
    ```
    
    *План выполнения:*
    ![image](https://github.com/user-attachments/assets/25a1ca1d-a66e-4380-b772-e7ac2a96688f)
    
    *Объясните результат:*
    1. HashAggregate по ключу *author*.
    2. $Batch$. Использование памяти - *193* Кбайт.
    3. Последовательное сканирование (без индекса).
    4. Итог - очень долго по времени - $214$ мсек.

22. Выберите первых 10 авторов:
    ```sql
    EXPLAIN ANALYZE
    SELECT DISTINCT author 
    FROM t_books 
    ORDER BY author 
    LIMIT 10;
    ```
    
    *План выполнения:*
    ![image](https://github.com/user-attachments/assets/54ae0d0f-6568-4cbe-bd7b-397fbddd980d)

    
    *Объясните результат:*
    Сканирование по *Limit* - фактическое время $8.680..9.860$ мсек.

    Запрос на **уникальных** авторов. Фактическое время выполнения - $8.676..9.849$ мсек.

    Сканирование только по индексу $t_books_author_title_index$.

    Итого время выполнения - **более 10** миллисекунд. Ожидаемое время выполнения - менее $1$ миллисекунды.

    **Итого: довольно-таки долго**

24. Выполните поиск и сортировку:
    ```sql
    EXPLAIN ANALYZE
    SELECT author, title 
    FROM t_books 
    WHERE author LIKE 'T%'
    ORDER BY author, title;
    ```
    
    *План выполнения:*

    ![image](https://github.com/user-attachments/assets/50b78443-cc05-4b75-b8be-4d9bb866d804)

    
    *Объясните результат:*

    1. Происходит сортировка по парному ключу $(author, title)$. Метод сортировки $quicksort$.
    2. Последовательное сканирование с использованием фильтра на автора.
    3. Итого: фактические временные затраты - $31.895 ms$ (долго по времени), при ожидаемых около $3.1 ms$. Затраты на *cost* - $3100.29..3100.33$.

25. Добавьте новую книгу:
    ```sql
    INSERT INTO t_books (book_id, title, author, category, is_active)
    VALUES (150001, 'Cookbook', 'Mr. Hide', NULL, true);
    COMMIT;
    ```
    
    *Результат:*
    ![image](https://github.com/user-attachments/assets/14fe838f-5561-48ec-88de-40b98359a8e6)


26. Создайте индекс по категории:
    ```sql
    CREATE INDEX t_books_cat_idx ON t_books(category);
    ```
    
    *Результат:*
    
    ![image](https://github.com/user-attachments/assets/d4feae89-ee8c-4607-913b-a598225124ff)


28. Найдите книги без категории:
    ```sql
    EXPLAIN ANALYZE
    SELECT author, title 
    FROM t_books 
    WHERE category IS NULL;
    ```
    
    *План выполнения:*
    
    ![image](https://github.com/user-attachments/assets/00c39587-9406-4c89-bab3-deb5713ff477)

    
    *Объясните результат:*

    Сканирование по индексу на $category$ (при условии на индекс). Временные затраты на сканирование: ок. $0.1$ ms **меньше** ожидаемых: ок $0.7$ мсек. $Cost$ сравнительно небольшой - $8.15$, по сравнению с некоторыми операциями.

    **Итого: быстро и эффективно**

30. Создайте частичные индексы:
    ```sql
    DROP INDEX t_books_cat_idx;
    CREATE INDEX t_books_cat_null_idx ON t_books(category) WHERE category IS NULL;
    ```
    
    *Результат:*
    ![image](https://github.com/user-attachments/assets/ade9fc7d-c16b-465d-a0dc-b4904c91bcf6)


31. Повторите запрос из шага 22:
    ```sql
    EXPLAIN ANALYZE
    SELECT author, title 
    FROM t_books 
    WHERE category IS NULL;
    ```
    
    *План выполнения:*
    
    ![image](https://github.com/user-attachments/assets/64c8b734-5431-4a57-a6e7-46914034d92b)

    
    *Объясните результат:*
    
    Сканирование по индексу. Временные затраты : $0.07$ мсек, при ожидаемых $0.877$ мсек. Временные затраты **ещё меньше**, чем в предыдущем запросе. 

33. Создайте частичный уникальный индекс:
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
    [Вставьте результаты всех операций]
    
    *Объясните результат:*
    [Ваше объяснение]
