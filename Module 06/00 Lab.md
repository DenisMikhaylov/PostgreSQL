# Лабораторная работа: Индексы и планы выполнения в PostgreSQL


## Часть 0. Подготовка окружения (15 минут)

### 0.1. Создание базы данных

```sql
CREATE DATABASE lab_indexes;

\c lab_indexes
```

### 0.2. Включение расширений

```sql
CREATE EXTENSION IF NOT EXISTS pg_stat_statements;
CREATE EXTENSION IF NOT EXISTS bloom;
CREATE EXTENSION IF NOT EXISTS btree_gin;
CREATE EXTENSION IF NOT EXISTS btree_gist;
```

### 0.3. Создание тестовых таблиц

```sql
-- Таблица пользователей
CREATE TABLE users (
    user_id    SERIAL PRIMARY KEY,
    email      TEXT NOT NULL,
    username   TEXT NOT NULL,
    full_name  TEXT,
    age        INTEGER,
    city       TEXT,
    status     TEXT DEFAULT 'active',
    created_at TIMESTAMP DEFAULT NOW()
);

-- Таблица заказов
CREATE TABLE orders (
    order_id   SERIAL PRIMARY KEY,
    user_id    INTEGER NOT NULL,
    order_date DATE NOT NULL,
    amount     NUMERIC(10,2) NOT NULL,
    status     TEXT NOT NULL DEFAULT 'new'
);

-- Таблица товаров
CREATE TABLE products (
    product_id SERIAL PRIMARY KEY,
    name       TEXT NOT NULL,
    tags       TEXT[],
    attributes JSONB,
    price      NUMERIC(10,2),
    created_at TIMESTAMP
);
```

### 0.4. Заполнение тестовыми данными

```sql
-- 500 000 пользователей
INSERT INTO users (email, username, full_name, age, city, status)
SELECT 
    'user' || g || '@example.com',
    'user_' || g,
    'User Name ' || g,
    18 + (g % 60),
    (ARRAY['Москва','Санкт-Петербург','Казань','Новосибирск','Екатеринбург'])[1 + (g % 5)],
    (ARRAY['active','inactive','blocked'])[1 + (g % 3)]
FROM generate_series(1, 500000) AS g;

-- 1 000 000 заказов
INSERT INTO orders (user_id, order_date, amount, status)
SELECT 
    1 + (random() * 499999)::int,
    DATE '2023-01-01' + (random() * 700)::int,
    round((random() * 10000 + 100)::numeric, 2),
    (ARRAY['new','paid','shipped','delivered','cancelled'])[1 + floor(random()*5)::int]
FROM generate_series(1, 1000000);

-- 100 000 товаров
INSERT INTO products (name, tags, attributes, price, created_at)
SELECT 
    'Product ' || g,
    ARRAY[
        (ARRAY['electronics','clothing','books','food','toys'])[1 + (g % 5)],
        (ARRAY['sale','new','popular','premium'])[1 + (g % 4)]
    ],
    jsonb_build_object(
        'brand', (ARRAY['Apple','Samsung','Sony','LG','Xiaomi'])[1 + (g % 5)],
        'color', (ARRAY['black','white','red','blue'])[1 + (g % 4)],
        'in_stock', (g % 3 = 0)
    ),
    round((random() * 50000 + 500)::numeric, 2),
    NOW() - (g || ' hours')::interval
FROM generate_series(1, 100000) AS g;
```

### 0.5. Обновление статистики

```sql
ANALYZE users;
ANALYZE orders;
ANALYZE products;

-- Проверка размеров
SELECT 
    relname, 
    pg_size_pretty(pg_total_relation_size(oid)) AS total_size
FROM pg_class
WHERE relname IN ('users','orders','products')
ORDER BY relname;
```


## Часть 1. Основы EXPLAIN

### 1.1. Базовый EXPLAIN

**Задание 1.1:** Получите план простого запроса без индексов.

```sql
EXPLAIN SELECT * FROM users WHERE email = 'user12345@example.com';
```

**Вопрос:** Какой тип сканирования используется? Почему?

**Задание 1.2:** Используйте `EXPLAIN ANALYZE` для получения фактических метрик.

```sql
EXPLAIN (ANALYZE, BUFFERS, TIMING) 
SELECT * FROM users WHERE email = 'user12345@example.com';
```

**Что смотреть в выводе:**
- `cost=0.00..XXXXX.XX` — оценка стоимости (начальная..общая)
- `rows=N` — ожидаемое количество строк
- `width=N` — средняя ширина строки
- `actual time=X..Y rows=N loops=N` — фактические метрики
- `Buffers: shared hit=N read=N` — использование кеша

**Задание 1.3:** Сравните оценку `rows` с фактическим значением. Если они сильно расходятся — обновите статистику.

### 1.2. Форматы вывода EXPLAIN

**Задание 1.4:** Получите план в разных форматах.

```sql
-- Текстовый формат (по умолчанию)
EXPLAIN (FORMAT TEXT) SELECT * FROM users WHERE age = 30;

-- JSON формат (удобен для программной обработки)
EXPLAIN (FORMAT JSON) SELECT * FROM users WHERE age = 30;

-- YAML формат
EXPLAIN (FORMAT YAML) SELECT * FROM users WHERE age = 30;
```

### 1.3. Анализ плана через pg_stat_statements

**Задание 1.5:** Проверьте, что ваши запросы записались в статистику.

```sql
SELECT queryid, calls, total_time, mean_time, left(query, 80)
FROM pg_stat_statements
WHERE query LIKE '%users%'
ORDER BY total_time DESC
LIMIT 5;
```


## Часть 2. B-tree индексы и их влияние на планы 

### 2.1. Seq Scan vs Index Scan

**Задание 2.1:** Создайте B-tree индекс на `email` и сравните планы.

```sql
-- Без индекса
EXPLAIN (ANALYZE, BUFFERS) 
SELECT * FROM users WHERE email = 'user12345@example.com';

-- Создаём индекс
CREATE INDEX idx_users_email ON users(email);

-- С индексом
EXPLAIN (ANALYZE, BUFFERS) 
SELECT * FROM users WHERE email = 'user12345@example.com';
```

**Сравните:**
- Сколько страниц читалось в первом и втором случае (`Buffers: shared read=N`)?
- Время выполнения?
- Тип сканирования?

**Задание 2.2:** Проверьте, что индекс ускоряет поиск по диапазону.

```sql
-- Поиск по диапазону дат (индекс на created_at)
CREATE INDEX idx_users_created_at ON users(created_at);

EXPLAIN (ANALYZE, BUFFERS)
SELECT * FROM users 
WHERE created_at > NOW() - INTERVAL '1 day';

-- Проверьте, что индекс не используется для запросов с функцией от столбца
EXPLAIN (ANALYZE, BUFFERS)
SELECT * FROM users 
WHERE date_trunc('day', created_at) = CURRENT_DATE;
```


### 2.2. Bitmap Heap Scan

**Задание 2.3:** Спровоцируйте Bitmap Heap Scan — выберите средний процент строк.

```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT * FROM users WHERE age BETWEEN 30 AND 45;
```

**Попробуйте разные диапазоны:**
- Маленький (< 1% строк) → обычно Index Scan
- Средний (5–20% строк) → обычно Bitmap Heap Scan
- Большой (> 30% строк) → обычно Seq Scan

```sql
-- Создаём индекс для теста
CREATE INDEX idx_users_age ON users(age);

-- Узкий диапазон (Index Scan)
EXPLAIN (ANALYZE, BUFFERS) SELECT * FROM users WHERE age = 30;

-- Средний диапазон (Bitmap Heap Scan)
EXPLAIN (ANALYZE, BUFFERS) SELECT * FROM users WHERE age BETWEEN 25 AND 35;

-- Широкий диапазон (Seq Scan)
EXPLAIN (ANALYZE, BUFFERS) SELECT * FROM users WHERE age BETWEEN 18 AND 80;
```


### 2.3. Влияние random_page_cost

**Задание 2.4:** Поэкспериментируйте с параметром `random_page_cost`.

```sql
-- Текущее значение
SHOW random_page_cost;  -- по умолчанию 4.0

-- Увеличим стоимость случайного чтения — планировщик будет избегать Index Scan
SET random_page_cost = 8;
EXPLAIN (ANALYZE, BUFFERS) SELECT * FROM users WHERE age = 30;

-- Уменьшим — планировщик будет предпочитать Index Scan
SET random_page_cost = 1.1;
EXPLAIN (ANALYZE, BUFFERS) SELECT * FROM users WHERE age = 30;

-- Возврат к значению по умолчанию
RESET random_page_cost;
```


## Часть 3. Составные индексы

### 3.1. Принцип работы составного индекса

**Задание 3.1:** Создайте составной индекс и проверьте его использование при разных условиях.

```sql
CREATE INDEX idx_users_city_status_age ON users(city, status, age);

-- Условие по всем столбцам — индекс используется максимально эффективно
EXPLAIN (ANALYZE, BUFFERS)
SELECT * FROM users 
WHERE city = 'Москва' AND status = 'active' AND age = 30;

-- Условие только по ведущему столбцу
EXPLAIN (ANALYZE, BUFFERS)
SELECT * FROM users WHERE city = 'Москва';

-- Условие только по второму столбцу (индекс НЕ используется эффективно)
EXPLAIN (ANALYZE, BUFFERS)
SELECT * FROM users WHERE status = 'active';

-- Условие только по третьему столбцу (индекс НЕ используется)
EXPLAIN (ANALYZE, BUFFERS)
SELECT * FROM users WHERE age = 30;
```

### 3.2. Оптимальный порядок столбцов

**Задание 3.2:** Создайте два индекса с разным порядком столбцов и сравните их эффективность.

```sql
-- Индекс 1: (status, city)
CREATE INDEX idx_users_status_city ON users(status, city);

-- Индекс 2: (city, status)
CREATE INDEX idx_users_city_status ON users(city, status);

-- Для запроса с равенствами по обоим столбцам оба индекса работают
EXPLAIN (ANALYZE, BUFFERS)
SELECT * FROM users WHERE status = 'active' AND city = 'Москва';

-- Но для запроса только по city эффективен только второй индекс
EXPLAIN (ANALYZE, BUFFERS)
SELECT * FROM users WHERE city = 'Москва';
```


### 3.3. Влияние количества строк

**Задание 3.3:** Проверьте, как индекс помогает при выборке разных объёмов данных.

```sql
-- 1 строка
EXPLAIN (ANALYZE, BUFFERS)
SELECT * FROM users WHERE city = 'Москва' AND status = 'blocked' AND age = 30;

-- 10 000 строк (20% от 'Москва' × 'active')
EXPLAIN (ANALYZE, BUFFERS)
SELECT * FROM users WHERE city = 'Москва' AND status = 'active';
```


## Часть 4. Частичные (фильтрованные) индексы 

### 4.1. Создание и использование

**Задание 4.1:** Создайте частичный индекс для активных пользователей.

```sql
-- Полный индекс
CREATE INDEX idx_users_status_full ON users(status);

-- Частичный индекс (только active)
CREATE INDEX idx_users_status_active ON users(status) WHERE status = 'active';

-- Сравните размеры
SELECT 
    indexrelname, 
    pg_size_pretty(pg_relation_size(indexrelid)) AS size
FROM pg_stat_user_indexes
WHERE relname = 'users' AND indexrelname LIKE '%status%';
```


**Задание 4.2:** Проверьте, когда планировщик использует частичный индекс.

```sql
-- Запрос, соответствующий условию индекса — используется
EXPLAIN (ANALYZE, BUFFERS)
SELECT * FROM users WHERE status = 'active' AND city = 'Москва';

-- Запрос НЕ соответствующий условию — не используется
EXPLAIN (ANALYZE, BUFFERS)
SELECT * FROM users WHERE status = 'inactive';
```

### 4.2. Частичный индекс для "горячих" данных

**Задание 4.3:** Создайте частичный индекс для неоплаченных заказов.

```sql
-- Полный индекс на status
CREATE INDEX idx_orders_status ON orders(status);

-- Частичный индекс только для новых заказов (которых мало, но они часто запрашиваются)
CREATE INDEX idx_orders_status_new ON orders(status) WHERE status = 'new';

-- Сравните размеры
SELECT 
    indexrelname, 
    pg_size_pretty(pg_relation_size(indexrelid)) AS size
FROM pg_stat_user_indexes
WHERE relname = 'orders' AND indexrelname LIKE '%status%';
```

### 4.3. Практическое задание

**Задание 4.4:** Создайте частичный индекс для заказов за последние 30 дней.

```sql
CREATE INDEX idx_orders_recent ON orders(order_date) 
WHERE order_date > '2024-12-01';

-- Проверьте план для запроса, соответствующего условию
EXPLAIN (ANALYZE, BUFFERS)
SELECT * FROM orders 
WHERE order_date > '2024-12-15' 
  AND amount > 5000;
```


## Часть 5. Индексы на выражениях 

### 5.1. Проблема: функции блокируют индексы

**Задание 5.1:** Продемонстрируйте, что индекс не работает с функцией от столбца.

```sql
-- Обычный индекс на email
CREATE INDEX idx_users_email_plain ON users(email);

-- Работает: поиск по точному значению
EXPLAIN (ANALYZE, BUFFERS)
SELECT * FROM users WHERE email = 'user12345@example.com';

-- НЕ работает: поиск по нижнему регистру
EXPLAIN (ANALYZE, BUFFERS)
SELECT * FROM users WHERE lower(email) = 'user12345@example.com';
```


### 5.2. Создание индекса на выражении

**Задание 5.2:** Создайте индекс на `lower(email)` и повторите запрос.

```sql
CREATE INDEX idx_users_email_lower ON users(lower(email));

EXPLAIN (ANALYZE, BUFFERS)
SELECT * FROM users WHERE lower(email) = 'user12345@example.com';
```

**Задание 5.3:** Создайте индекс на выражении с конкатенацией.

```sql
CREATE INDEX idx_users_fullname_lower ON users(lower(full_name));

EXPLAIN (ANALYZE, BUFFERS)
SELECT * FROM users WHERE lower(full_name) = 'user name 12345';
```

### 5.3. Индекс на выражении с JSONB

**Задание 5.4:** Создайте индекс на выражении, извлекающем поле из JSONB.

```sql
-- Индекс на конкретное поле в JSONB
CREATE INDEX idx_products_brand ON products((attributes->>'brand'));

-- Проверка
EXPLAIN (ANALYZE, BUFFERS)
SELECT * FROM products WHERE attributes->>'brand' = 'Apple';

-- Сравните с запросом без индекса
DROP INDEX idx_products_brand;
EXPLAIN (ANALYZE, BUFFERS)
SELECT * FROM products WHERE attributes->>'brand' = 'Apple';
```

### 5.4. Ограничение: IMMUTABLE функции

**Задание 5.5:** Попробуйте создать индекс с нестабильной функцией.

```sql
-- ОШИБКА: NOW() не является IMMUTABLE
-- CREATE INDEX idx_users_created ON users((created_at + INTERVAL '1 day'));
-- ERROR: functions in index expression must be marked IMMUTABLE

-- Работает, если использовать IMMUTABLE функцию
CREATE INDEX idx_users_created_date ON users((created_at::date));
```


## Часть 6. Другие виды индексов

### 6.1. GIN — индексы для массивов и JSONB

**Задание 6.1:** Создайте GIN индекс для массива тегов.

```sql
-- Без индекса
EXPLAIN (ANALYZE, BUFFERS)
SELECT * FROM products WHERE tags @> ARRAY['sale'];

-- Создаём GIN индекс
CREATE INDEX idx_products_tags ON products USING GIN(tags);

-- С индексом
EXPLAIN (ANALYZE, BUFFERS)
SELECT * FROM products WHERE tags @> ARRAY['sale'];

-- Проверка размера индекса
SELECT pg_size_pretty(pg_relation_size('idx_products_tags'));
```

**Задание 6.2:** Создайте GIN индекс для JSONB.

```sql
-- GIN индекс для всех ключей и значений JSONB
CREATE INDEX idx_products_attributes ON products USING GIN(attributes);

-- Поиск по любому ключу
EXPLAIN (ANALYZE, BUFFERS)
SELECT * FROM products WHERE attributes @> '{"brand": "Apple"}';

-- Поиск по вложенному ключу
EXPLAIN (ANALYZE, BUFFERS)
SELECT * FROM products WHERE attributes @> '{"color": "black", "in_stock": true}';
```

**Задание 6.3:** Создайте GIN индекс для полнотекстового поиска.

```sql
-- Добавим столбец tsvector
ALTER TABLE products ADD COLUMN search_vector tsvector;

UPDATE products 
SET search_vector = to_tsvector('russian', name);

-- GIN индекс
CREATE INDEX idx_products_search ON products USING GIN(search_vector);

-- Полнотекстовый поиск
EXPLAIN (ANALYZE, BUFFERS)
SELECT * FROM products 
WHERE search_vector @@ to_tsquery('russian', 'Product & 12345');
```





## Часть 7. Планы соединений (JOIN)

### 7.1. Nested Loop Join

**Задание 7.1:** Спровоцируйте Nested Loop и изучите его план.

```sql
-- Запрос с малым количеством строк во внешней таблице
EXPLAIN (ANALYZE, BUFFERS)
SELECT u.user_id, u.username, o.order_id, o.amount
FROM users u
JOIN orders o ON u.user_id = o.user_id
WHERE u.username = 'user_12345';
```

**Что смотреть:**
- Внешний цикл (`Nested Loop`)
- Внутренний `Index Scan` по индексу на `orders.user_id`
- Количество `loops` — сколько раз выполнялся внутренний цикл

**Задание 7.2:** Убедитесь, что индекс на внешнем ключе критичен для Nested Loop.

```sql
-- Индекс на orders.user_id
CREATE INDEX idx_orders_user_id ON orders(user_id);

EXPLAIN (ANALYZE, BUFFERS)
SELECT u.user_id, u.username, o.order_id
FROM users u
JOIN orders o ON u.user_id = o.user_id
WHERE u.username = 'user_12345';

-- Удалите индекс и повторите
DROP INDEX idx_orders_user_id;

EXPLAIN (ANALYZE, BUFFERS)
SELECT u.user_id, u.username, o.order_id
FROM users u
JOIN orders o ON u.user_id = o.user_id
WHERE u.username = 'user_12345';
```


### 7.2. Hash Join

**Задание 7.3:** Спровоцируйте Hash Join.

```sql
-- Восстанавливаем индекс
CREATE INDEX idx_orders_user_id ON orders(user_id);

-- Запрос с большим количеством строк
EXPLAIN (ANALYZE, BUFFERS)
SELECT u.city, count(o.order_id), sum(o.amount)
FROM users u
JOIN orders o ON u.user_id = o.user_id
GROUP BY u.city;
```

**Что смотреть:**
- Узел `Hash Join` с `Hash Cond`
- Узел `Hash` — строит хеш-таблицу
- Сколько памяти использовано (`Batches`, `Memory Usage`)

**Задание 7.4:** Поэкспериментируйте с `work_mem`.

```sql
-- Установим маленький work_mem
SET work_mem = '1MB';
EXPLAIN (ANALYZE, BUFFERS)
SELECT u.city, count(o.order_id)
FROM users u
JOIN orders o ON u.user_id = o.user_id
GROUP BY u.city;

-- Установим большой work_mem
SET work_mem = '256MB';
EXPLAIN (ANALYZE, BUFFERS)
SELECT u.city, count(o.order_id)
FROM users u
JOIN orders o ON u.user_id = o.user_id
GROUP BY u.city;

RESET work_mem;
```

### 7.3. Merge Join

**Задание 7.5:** Спровоцируйте Merge Join.

```sql
-- Убедимся, что есть индексы на столбцах соединения
CREATE INDEX IF NOT EXISTS idx_users_user_id ON users(user_id);
CREATE INDEX IF NOT EXISTS idx_orders_user_id ON orders(user_id);

-- Отключаем Hash Join и Nested Loop, чтобы заставить планировщик использовать Merge Join
SET enable_hashjoin = off;
SET enable_nestloop = off;

EXPLAIN (ANALYZE, BUFFERS)
SELECT u.user_id, u.username, o.order_id
FROM users u
JOIN orders o ON u.user_id = o.user_id
LIMIT 100;

-- Возвращаем настройки
RESET enable_hashjoin;
RESET enable_nestloop;
```

**Что смотреть:**
- Узел `Merge Join` с `Merge Cond`
- Наличие узлов `Sort` (если данные не отсортированы)

### 7.4. Сравнение алгоритмов соединения

**Задание 7.6:** Выполните один и тот же запрос с разными настройками и сравните время.

```sql
-- Функция для замера времени
CREATE OR REPLACE FUNCTION benchmark(query TEXT, iterations INT DEFAULT 5)
RETURNS TABLE(algorithm TEXT, avg_time_ms NUMERIC) AS $$
DECLARE
    i INT;
    start_time TIMESTAMPTZ;
    total_time NUMERIC := 0;
BEGIN
    FOR i IN 1..iterations LOOP
        start_time := clock_timestamp();
        EXECUTE query;
        total_time := total_time + EXTRACT(MILLISECONDS FROM clock_timestamp() - start_time);
    END LOOP;
    RETURN QUERY SELECT 'measured', round(total_time / iterations, 3);
END;
$$ LANGUAGE plpgsql;

-- Принудительно Nested Loop
SET enable_hashjoin = off;
SET enable_mergejoin = off;
SELECT * FROM benchmark($$SELECT u.city, count(o.order_id)
FROM users u JOIN orders o ON u.user_id = o.user_id GROUP BY u.city$$);

-- Принудительно Hash Join
SET enable_nestloop = off;
SET enable_mergejoin = off;
SELECT * FROM benchmark($$SELECT u.city, count(o.order_id)
FROM users u JOIN orders o ON u.user_id = o.user_id GROUP BY u.city$$);

-- Принудительно Merge Join
SET enable_nestloop = off;
SET enable_hashjoin = off;
SELECT * FROM benchmark($$SELECT u.city, count(o.order_id)
FROM users u JOIN orders o ON u.user_id = o.user_id GROUP BY u.city$$);

RESET ALL;
```




## Очистка окружения

```sql
-- Удаление базы данных (все объекты удалятся автоматически)
\c postgres
DROP DATABASE IF EXISTS lab_indexes;
```

**Успешной работы!**
