# Лабораторная работа: Настройка планировщика, статистика и оптимизация запросов


**Цель работы:** Научиться настраивать параметры планировщика (`random_page_cost`, `work_mem`, `effective_cache_size`), анализировать и корректировать статистику (`ANALYZE`, `autovacuum`, `default_statistics_target`), управлять соединениями и коррелированными подзапросами, а также использовать партиционирование и `constraint_exclusion` для оптимизации запросов.

---

## Часть 0. Подготовка окружения

### 0.1. Создание базы данных

**Что делаем:** Создаём отдельную БД для экспериментов.

**Почему:** Параметры планировщика и статистика зависят от данных. Изолированная БД даёт контроль над условиями.

```sql
CREATE DATABASE lab_optimizer;

\c lab_optimizer
```

### 0.2. Создание таблиц

```sql
-- Клиенты
CREATE TABLE customers (
    customer_id SERIAL PRIMARY KEY,
    name        TEXT NOT NULL,
    email       TEXT NOT NULL,
    city        TEXT NOT NULL,
    status      TEXT NOT NULL DEFAULT 'active',
    created_at  TIMESTAMP DEFAULT NOW()
);

-- Товары
CREATE TABLE products (
    product_id SERIAL PRIMARY KEY,
    name       TEXT NOT NULL,
    category   TEXT NOT NULL,
    price      NUMERIC(10,2) NOT NULL,
    stock      INTEGER DEFAULT 0
);

-- Заказы (большая таблица)
CREATE TABLE orders (
    order_id    SERIAL PRIMARY KEY,
    customer_id INTEGER NOT NULL,
    product_id  INTEGER NOT NULL,
    quantity    INTEGER NOT NULL,
    amount      NUMERIC(12,2) NOT NULL,
    order_date  DATE NOT NULL,
    status      TEXT NOT NULL DEFAULT 'new'
);
```

### 0.3. Заполнение данными

```sql
-- 50 000 клиентов
INSERT INTO customers (name, email, city, status, created_at)
SELECT 
    'Customer ' || g,
    'customer' || g || '@example.com',
    (ARRAY['Москва','Санкт-Петербург','Казань','Новосибирск','Екатеринбург'])[1 + (g % 5)],
    (ARRAY['active','inactive','blocked'])[1 + (g % 3)],
    NOW() - (g || ' minutes')::interval
FROM generate_series(1, 50000) AS g;

-- 500 товаров
INSERT INTO products (name, category, price, stock)
SELECT 
    'Product ' || g,
    (ARRAY['electronics','clothing','books','food','toys'])[1 + (g % 5)],
    round((random() * 5000 + 100)::numeric, 2),
    (random() * 1000)::int
FROM generate_series(1, 500) AS g;

-- 1 000 000 заказов
INSERT INTO orders (customer_id, product_id, quantity, amount, order_date, status)
SELECT 
    1 + (random() * 49999)::int,
    1 + (random() * 499)::int,
    1 + (random() * 10)::int,
    round((random() * 10000 + 100)::numeric, 2),
    DATE '2024-01-01' + (random() * 365)::int,
    (ARRAY['new','paid','shipped','delivered','cancelled'])[1 + floor(random()*5)::int]
FROM generate_series(1, 1000000);
```

### 0.4. Сбор статистики

```sql
ANALYZE customers;
ANALYZE products;
ANALYZE orders;

-- Проверка размеров
SELECT 
    relname,
    relpages,
    reltuples::bigint AS approx_rows,
    pg_size_pretty(pg_total_relation_size(oid)) AS total_size
FROM pg_class
WHERE relname IN ('customers','products','orders')
ORDER BY relname;
```


## Часть 1. Параметр random_page_cost

### 1.1. Что такое random_page_cost?

**Что делаем:** Смотрим текущее значение.

**Почему:** `random_page_cost` определяет «стоимость» случайного чтения страницы. От него зависит выбор между `Seq Scan` и `Index Scan`.

```sql
SHOW random_page_cost;   -- по умолчанию 4.0 (для HDD)
SHOW seq_page_cost;      -- всегда 1.0
```

### 1.2. Влияние на план

**Что делаем:** Выполняем запрос при разных значениях.

**Почему:** Покажем, что значение `random_page_cost` управляет предпочтением индексов.

```sql
CREATE INDEX idx_orders_amount ON orders(amount);

-- HDD (по умолчанию)
SET random_page_cost = 4.0;
EXPLAIN (ANALYZE, COSTS OFF)
SELECT * FROM orders WHERE amount BETWEEN 8000 AND 9000;

-- SSD
SET random_page_cost = 1.1;
EXPLAIN (ANALYZE, COSTS OFF)
SELECT * FROM orders WHERE amount BETWEEN 8000 AND 9000;

RESET random_page_cost;
```

**Что смотреть:**
- При `4.0` — вероятно `Seq Scan` (случайные чтения дорогие).
- При `1.1` — вероятно `Bitmap Heap Scan` или `Index Scan`.

### 1.3. Практическое задание

**Задание 1.3:** Определите значение `random_page_cost`, при котором план меняется с `Seq Scan` на `Index/Bitmap Scan`.

```sql
-- Перебирайте значения: 4.0, 3.0, 2.0, 1.5, 1.1
SET random_page_cost = 2.0;
EXPLAIN (ANALYZE, COSTS OFF)
SELECT * FROM orders WHERE amount BETWEEN 8000 AND 9000;
```

## Часть 2. Параметр work_mem

### 2.1. Влияние на сортировку

**Что делаем:** Сортируем большую таблицу при разных `work_mem`.

**Почему:** Покажем, что при нехватке памяти сортировка выгружается на диск.

```sql
-- Маленький work_mem
SET work_mem = '1MB';
EXPLAIN (ANALYZE, BUFFERS)
SELECT * FROM orders ORDER BY amount DESC LIMIT 100;

-- Большой work_mem
SET work_mem = '128MB';
EXPLAIN (ANALYZE, BUFFERS)
SELECT * FROM orders ORDER BY amount DESC LIMIT 100;

RESET work_mem;
```

**Что смотреть в узле `Sort`:**

| Метрика | `work_mem = 1MB` | `work_mem = 128MB` |
|---------|------------------|---------------------|
| `Sort Method` | `external merge` (диск) | `quicksort` (память) |
| `Disk: NkB` | Есть | Нет |
| `Execution Time` | Больше | Меньше |

### 2.2. Влияние на Hash Join

**Что делаем:** Соединяем две таблицы при разных `work_mem`.

**Почему:** Hash Join требует памяти для хеш-таблицы. При нехватке — временные файлы.

```sql
SET work_mem = '1MB';
EXPLAIN (ANALYZE, BUFFERS)
SELECT c.city, count(*), sum(o.amount)
FROM customers c
JOIN orders o ON c.customer_id = o.customer_id
GROUP BY c.city;

SET work_mem = '256MB';
EXPLAIN (ANALYZE, BUFFERS)
SELECT c.city, count(*), sum(o.amount)
FROM customers c
JOIN orders o ON c.customer_id = o.customer_id
GROUP BY c.city;

RESET work_mem;
```

**Что смотреть в узле `Hash`:**

| Метрика | Маленький `work_mem` | Большой `work_mem` |
|---------|----------------------|---------------------|
| `Batches` | > 1 (например, 8) | 1 |
| `Memory Usage` | Малое | Большое |
| `Disk Usage` | Есть | Нет |
| `Execution Time` | Больше | Меньше |

### 2.3. Практическое задание

**Задание 2.3:** Найдите минимальный `work_mem`, при котором `Batches = 1`.

```sql
SET work_mem = '32MB';
EXPLAIN (ANALYZE, BUFFERS)
SELECT c.city, count(*), sum(o.amount)
FROM customers c
JOIN orders o ON c.customer_id = o.customer_id
GROUP BY c.city;
```


## Часть 3. Параметр effective_cache_size

### 3.1. Влияние на выбор индекса

**Что делаем:** Сравниваем планы при разных `effective_cache_size`.

**Почему:** Чем выше значение, тем охотнее планировщик использует индексы.

```sql
-- Маленький кеш
SET effective_cache_size = '128MB';
EXPLAIN (ANALYZE, COSTS OFF)
SELECT * FROM orders WHERE amount > 9000;

-- Большой кеш
SET effective_cache_size = '24GB';
EXPLAIN (ANALYZE, COSTS OFF)
SELECT * FROM orders WHERE amount > 9000;

RESET effective_cache_size;
```

## Часть 4. Статистика: ANALYZE и autovacuum

### 4.1. Просмотр статистики

```sql
SELECT 
    attname AS column_name,
    null_frac,
    n_distinct,
    correlation
FROM pg_stats
WHERE tablename = 'orders'
ORDER BY attname;
```

**Что смотреть:**

| Столбец | `n_distinct` | `correlation` | Что это значит |
|---------|--------------|---------------|----------------|
| `order_id` | -1 | ~1.0 | Уникальный, физически упорядочен |
| `customer_id` | ~50000 | ~0.0 | Высокая кардинальность, случайный порядок |
| `status` | 5 | ~0.0 | Низкая кардинальность |

### 4.2. Имитация устаревшей статистики

**Что делаем:** Массово меняем данные, не запуская `ANALYZE`.

**Почему:** Покажем, что планировщик строит плохие планы на устаревшей статистике.

```sql
-- План ДО изменения
EXPLAIN (ANALYZE, COSTS OFF)
SELECT * FROM orders WHERE status = 'paid';

-- Массово меняем данные
UPDATE orders SET status = 'paid' WHERE status = 'new';

-- План СРАЗУ после UPDATE (без ANALYZE)
EXPLAIN (ANALYZE, COSTS OFF)
SELECT * FROM orders WHERE status = 'paid';
```

**Что смотреть:** `rows` в оценке может сильно отличаться от `actual rows`.

```sql
-- Обновляем статистику
ANALYZE orders;

-- План ПОСЛЕ ANALYZE
EXPLAIN (ANALYZE, COSTS OFF)
SELECT * FROM orders WHERE status = 'paid';
```


### 4.3. Настройка autovacuum для таблицы

**Что делаем:** Уменьшаем порог запуска autovacuum.

**Почему:** Для больших таблиц стандартный порог (10%) слишком велик.

```sql
SHOW autovacuum_analyze_threshold;         -- 50
SHOW autovacuum_analyze_scale_factor;      -- 0.1

-- Настройка для конкретной таблицы
ALTER TABLE orders SET (
    autovacuum_analyze_scale_factor = 0.01,
    autovacuum_analyze_threshold = 100
);

-- Проверка
SELECT relname, reloptions
FROM pg_class
WHERE relname = 'orders';
```

### 4.4. Просмотр свежести статистики

```sql
SELECT 
    relname,
    last_analyze,
    last_autoanalyze,
    n_mod_since_analyze,
    n_live_tup
FROM pg_stat_user_tables
WHERE schemaname = 'public'
ORDER BY n_mod_since_analyze DESC;
```


## Часть 5. default_statistics_target

### 5.1. Что это?

**Что делаем:** Смотрим текущее значение.

**Почему:** Это глобальная «глубина» статистики. Чем больше — тем точнее оценки, но дольше `ANALYZE` и больше места.

```sql
SHOW default_statistics_target;  -- обычно 100
```

### 5.2. Влияние на MCV и гистограмму

```sql
-- Размер MCV и гистограммы для city
SELECT 
    attname,
    array_length(most_common_vals::text::text[], 1) AS mcv_size,
    array_length(histogram_bounds::text::text[], 1) AS histogram_size
FROM pg_stats
WHERE tablename = 'customers' AND attname = 'city';
```

### 5.3. Увеличение точности для столбца

```sql
-- Повышаем точность для city
ALTER TABLE customers ALTER COLUMN city SET STATISTICS 500;

-- Пересобираем статистику
ANALYZE customers;

-- Проверяем
SELECT 
    attname,
    array_length(most_common_vals::text::text[], 1) AS mcv_size,
    array_length(histogram_bounds::text::text[], 1) AS histogram_size
FROM pg_stats
WHERE tablename = 'customers' AND attname = 'city';
```


### 5.4. Просмотр индивидуальных настроек

```sql
SELECT attname, attstattarget
FROM pg_attribute
WHERE attrelid = 'customers'::regclass
  AND attnum > 0
ORDER BY attname;
```

### 5.5. Практическое задание

**Задание 5.5:** Найдите столбец с самым неравномерным распределением и увеличьте для него `STATISTICS` до 1000.

```sql
SELECT attname, most_common_vals, most_common_freqs
FROM pg_stats
WHERE tablename = 'orders'
  AND attname IN ('status', 'order_date');
```


## Часть 6. Соединения и коррелированные подзапросы

### 6.1. Коррелированный подзапрос vs JOIN

**Что делаем:** Сравниваем два эквивалентных запроса.

```sql
-- Вариант 1: коррелированный подзапрос
EXPLAIN (ANALYZE, BUFFERS, COSTS OFF)
SELECT 
    c.customer_id,
    c.name,
    (SELECT count(*) FROM orders o WHERE o.customer_id = c.customer_id) AS order_count
FROM customers c
WHERE c.city = 'Москва';

-- Вариант 2: JOIN с группировкой
EXPLAIN (ANALYZE, BUFFERS, COSTS OFF)
SELECT 
    c.customer_id,
    c.name,
    count(o.order_id) AS order_count
FROM customers c
LEFT JOIN orders o ON o.customer_id = c.customer_id
WHERE c.city = 'Москва'
GROUP BY c.customer_id, c.name;
```

**Что смотреть:**
- В первом случае может использоваться `SubPlan` — выполняется **для каждой строки**.
- Во втором — `Hash Join` или `Nested Loop` с одним проходом.

### 6.2. EXISTS vs IN

```sql
-- EXISTS
EXPLAIN (ANALYZE, BUFFERS, COSTS OFF)
SELECT * FROM customers c
WHERE EXISTS (
    SELECT 1 FROM orders o WHERE o.customer_id = c.customer_id
);

-- IN
EXPLAIN (ANALYZE, BUFFERS, COSTS OFF)
SELECT * FROM customers c
WHERE c.customer_id IN (
    SELECT customer_id FROM orders
);
```

**Что смотреть:**
- `EXISTS` часто преобразуется в `Hash Semi Join` — эффективно.
- `IN` может быть преобразован в `Hash Join` или `Nested Loop`.

### 6.3. Влияние статистики на выбор соединения

```sql
EXPLAIN (ANALYZE, BUFFERS, COSTS OFF)
SELECT 
    c.city,
    p.category,
    count(*),
    sum(o.amount)
FROM orders o
JOIN customers c ON o.customer_id = c.customer_id
JOIN products p ON o.product_id = p.product_id
WHERE c.city = 'Москва'
  AND p.category = 'electronics'
GROUP BY c.city, p.category;
```




## Часть 7. Партиционирование и constraint_exclusion 

### 7.1. Создание партиционированной таблицы

```sql
-- Основная таблица
CREATE TABLE events (
    event_id   SERIAL,
    event_date DATE NOT NULL,
    event_type TEXT NOT NULL,
    payload    JSONB
) PARTITION BY RANGE (event_date);

-- Секции по годам
CREATE TABLE events_2022 PARTITION OF events
    FOR VALUES FROM ('2022-01-01') TO ('2023-01-01');

CREATE TABLE events_2023 PARTITION OF events
    FOR VALUES FROM ('2023-01-01') TO ('2024-01-01');

CREATE TABLE events_2024 PARTITION OF events
    FOR VALUES FROM ('2024-01-01') TO ('2025-01-01');

CREATE TABLE events_2025 PARTITION OF events
    FOR VALUES FROM ('2025-01-01') TO ('2026-01-01');
```

### 7.2. Заполнение данными

```sql
INSERT INTO events (event_date, event_type, payload)
SELECT 
    DATE '2022-01-01' + (random() * 364)::int,
    'type_' || (1 + (random() * 10)::int),
    jsonb_build_object('value', random())
FROM generate_series(1, 100000);

INSERT INTO events (event_date, event_type, payload)
SELECT 
    DATE '2023-01-01' + (random() * 364)::int,
    'type_' || (1 + (random() * 10)::int),
    jsonb_build_object('value', random())
FROM generate_series(1, 200000);

INSERT INTO events (event_date, event_type, payload)
SELECT 
    DATE '2024-01-01' + (random() * 364)::int,
    'type_' || (1 + (random() * 10)::int),
    jsonb_build_object('value', random())
FROM generate_series(1, 300000);

ANALYZE events;
```

### 7.3. Демонстрация partition pruning

```sql
-- Включён partition pruning (по умолчанию)
SHOW enable_partition_pruning;

EXPLAIN (ANALYZE, COSTS OFF)
SELECT * FROM events 
WHERE event_date BETWEEN '2024-01-01' AND '2024-12-31';
```

**Что смотреть:** В плане только `events_2024`, а не все 4 секции.

**Отключаем pruning:**
```sql
SET enable_partition_pruning = off;
EXPLAIN (ANALYZE, COSTS OFF)
SELECT * FROM events 
WHERE event_date BETWEEN '2024-01-01' AND '2024-12-31';
RESET enable_partition_pruning;
```

**Что смотреть:** В плане **все** секции — оптимизация отключена.

### 7.4. Проблема с неиммутабельными функциями

```sql
EXPLAIN (ANALYZE, COSTS OFF)
SELECT * FROM events 
WHERE event_date >= CURRENT_DATE - INTERVAL '1 year';
```

**Что смотреть:** В плане **все** секции — `CURRENT_DATE` не константа на этапе планирования.

**Как исправить:**
```sql
EXPLAIN (ANALYZE, COSTS OFF)
SELECT * FROM events 
WHERE event_date >= '2024-01-01';
```

### 7.5. constraint_exclusion для наследуемых таблиц

```sql
-- Родительская таблица
CREATE TABLE legacy_events (
    event_id   SERIAL,
    event_date DATE NOT NULL,
    event_type TEXT NOT NULL
);

-- Дочерние таблицы с CHECK-ограничениями
CREATE TABLE legacy_events_2023 (
    CHECK (event_date >= '2023-01-01' AND event_date < '2024-01-01')
) INHERITS (legacy_events);

CREATE TABLE legacy_events_2024 (
    CHECK (event_date >= '2024-01-01' AND event_date < '2025-01-01')
) INHERITS (legacy_events);

-- Заполнение
INSERT INTO legacy_events_2023 (event_date, event_type)
SELECT DATE '2023-01-01' + (random() * 364)::int, 'type_a'
FROM generate_series(1, 50000);

INSERT INTO legacy_events_2024 (event_date, event_type)
SELECT DATE '2024-01-01' + (random() * 364)::int, 'type_b'
FROM generate_series(1, 50000);

ANALYZE legacy_events;
ANALYZE legacy_events_2023;
ANALYZE legacy_events_2024;
```

**Проверяем поведение:**

```sql
-- constraint_exclusion = off
SET constraint_exclusion = off;
EXPLAIN (ANALYZE, COSTS OFF)
SELECT * FROM legacy_events 
WHERE event_date >= '2024-01-01' AND event_date < '2025-01-01';

-- constraint_exclusion = on
SET constraint_exclusion = on;
EXPLAIN (ANALYZE, COSTS OFF)
SELECT * FROM legacy_events 
WHERE event_date >= '2024-01-01' AND event_date < '2025-01-01';

RESET constraint_exclusion;
```



## 🧹 Очистка окружения

```sql
-- Удаление таблиц
DROP TABLE IF EXISTS events CASCADE;
DROP TABLE IF EXISTS legacy_events CASCADE;
DROP TABLE IF EXISTS orders CASCADE;
DROP TABLE IF EXISTS customers CASCADE;
DROP TABLE IF EXISTS products CASCADE;

-- Сброс настроек сессии
RESET ALL;

-- Переключение и удаление базы
\c postgres
DROP DATABASE IF EXISTS lab_optimizer;
```

**Успешной работы!**
