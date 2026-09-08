# Лабораторная работа: Нормализация, типы данных, домены, наследование, партиционирование и идентификаторы



**Цель работы:** Изучить на практике ключевые аспекты проектирования и администрирования баз данных в PostgreSQL/Postgres Pro: нормализацию, использование различных типов данных, создание доменов и пользовательских типов, наследование таблиц, партиционирование с распределением по разным папкам, а также различные способы генерации идентификаторов.

---

## Часть 0. Подготовка окружения 

Прежде чем приступить к заданиям, создадим **отдельную базу данных** и **собственное табличное пространство** для изоляции лабораторной работы.

### 0.1. Создание каталога для табличных пространств

Выполните в терминале (от пользователя с правами sudo или от root):

```bash
# Создаём корневой каталог для всей лабораторной
sudo mkdir -p /tmp/lab_postgres
sudo chown postgres:postgres /tmp/lab_postgres
```

### 0.2. Создание табличного пространства и базы данных

Подключитесь к PostgreSQL под пользователем `postgres` и выполните:

```sql
-- Создаём табличное пространство для лабораторной
CREATE TABLESPACE lab_tablespace LOCATION '/tmp/lab_postgres';

-- Создаём базу данных, которая будет использовать это табличное пространство по умолчанию
CREATE DATABASE lab_db TABLESPACE lab_tablespace;

-- Подключаемся к новой базе данных
\c lab_db
```

> **Важно:** Все дальнейшие действия выполняются в базе данных `lab_db`. Это позволит не засорять системные базы и упростит очистку после работы.

### 0.3. Проверка текущего табличного пространства

Убедитесь, что вы работаете в правильной базе:

```sql
SELECT current_database(), current_schema();
```

Теперь все создаваемые объекты будут физически храниться в каталоге `/tmp/lab_postgres`.

---

## Часть 1. Нормализация базы данных (30 минут)

### 1.1. Анализ ненормализованной таблицы

Представьте, что вы проектируете базу данных для интернет-магазина. Рассмотрим **проблемную, ненормализованную** структуру:

```sql
-- НЕНОРМАЛИЗОВАННАЯ ТАБЛИЦА (проблемная структура)
CREATE TABLE orders_bad (
    order_id SERIAL PRIMARY KEY,
    customer_name VARCHAR(100),
    customer_phone VARCHAR(20),
    customer_email VARCHAR(100),
    order_date DATE,
    product_1_name VARCHAR(100),
    product_1_price DECIMAL(10,2),
    product_1_quantity INTEGER,
    product_2_name VARCHAR(100),
    product_2_price DECIMAL(10,2),
    product_2_quantity INTEGER,
    product_3_name VARCHAR(100),
    product_3_price DECIMAL(10,2),
    product_3_quantity INTEGER,
    total_amount DECIMAL(10,2)
);
```

**Задание 1.1:** Проанализируйте структуру таблицы `orders_bad` и выявите все проблемы с точки зрения нормализации. Запишите их в виде списка.


### 1.2. Проектирование нормализованной схемы

**Задание 1.2:** Разработайте нормализованную схему базы данных (3НФ) для интернет-магазина. Она должна включать таблицы для клиентов, заказов, товаров и позиций заказа.


---

## Часть 2. Типы данных, домены и пользовательские типы (35 минут)

### 2.1. Создание доменов

**Задание 2.1:** Создайте следующие домены для вашей базы данных:

```sql
-- 1. Домен для email-адреса (простая валидация)
CREATE DOMAIN valid_email AS VARCHAR(255)
    CHECK (VALUE ~* '^[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[A-Za-z]{2,}$');

-- 2. Домен для положительной цены
CREATE DOMAIN positive_price AS DECIMAL(10,2)
    CHECK (VALUE >= 0);

-- 3. Домен для номера телефона (российский формат)
CREATE DOMAIN russian_phone AS VARCHAR(20)
    CHECK (VALUE ~ '^\+7[0-9]{10}$' OR VALUE ~ '^8[0-9]{10}$');

-- 4. Домен для статуса заказа
CREATE DOMAIN order_status_type AS VARCHAR(20)
    DEFAULT 'new'
    CHECK (VALUE IN ('new', 'processing', 'shipped', 'delivered', 'cancelled'));

-- 5. Домен для неотрицательного количества
CREATE DOMAIN positive_quantity AS INTEGER
    CHECK (VALUE >= 0);
```

**Задание 2.2:** Создайте таблицы, используя созданные домены:

```sql
-- Таблица клиентов с доменами
CREATE TABLE customers (
    customer_id SERIAL PRIMARY KEY,
    name VARCHAR(100) NOT NULL,
    phone russian_phone NOT NULL,
    email valid_email UNIQUE,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);

-- Таблица товаров с доменами
CREATE TABLE products (
    product_id SERIAL PRIMARY KEY,
    name VARCHAR(200) NOT NULL,
    description TEXT,
    price positive_price NOT NULL,
    stock_quantity positive_quantity DEFAULT 0
);
```

**Задание 2.3:** Проверьте работу доменов — попробуйте вставить некорректные данные:

```sql
-- ОШИБКА: неверный формат телефона
INSERT INTO customers (name, phone, email) 
VALUES ('Иванов Иван', '89991234567', 'ivan@mail.ru');
-- ERROR: value for domain russian_phone violates check constraint

-- ОШИБКА: неверный формат email
INSERT INTO customers (name, phone, email) 
VALUES ('Иванов Иван', '+79991234567', 'ivan');

-- КОРРЕКТНО
INSERT INTO customers (name, phone, email) 
VALUES ('Иванов Иван', '+79991234567', 'ivan@mail.ru');
```

### 2.2. Создание составного пользовательского типа

**Задание 2.4:** Создайте составной тип для адреса:

```sql
-- Создание составного типа "address"
CREATE TYPE address_type AS (
    street VARCHAR(200),
    city VARCHAR(100),
    postal_code VARCHAR(20),
    country VARCHAR(100) DEFAULT 'Россия'
);

-- Использование составного типа в таблице
ALTER TABLE customers ADD COLUMN address address_type;
```

**Задание 2.5:** Вставьте данные с использованием составного типа:

```sql
-- Вставка с использованием составного типа
INSERT INTO customers (name, phone, email, address) 
VALUES (
    'Петров Пётр',
    '+79998887766',
    'petr@mail.ru',
    ROW('ул. Ленина, д. 10', 'Москва', '101000', 'Россия')::address_type
);

-- Запрос к полям составного типа
SELECT 
    name,
    phone,
    email,
    (address).street,
    (address).city,
    (address).postal_code
FROM customers;
```

### 2.3. Перечислимый тип (ENUM)

**Задание 2.6:** Создайте ENUM-тип для категорий товаров:

```sql
-- Создание ENUM-типа
CREATE TYPE product_category AS ENUM ('electronics', 'clothing', 'books', 'food', 'toys');

-- Добавление поля категории в таблицу продуктов
ALTER TABLE products ADD COLUMN category product_category;

-- Вставка данных с использованием ENUM
INSERT INTO products (name, price, stock_quantity, category) 
VALUES 
    ('Ноутбук', 50000, 10, 'electronics'),
    ('Футболка', 1000, 50, 'clothing'),
    ('Книга "Война и мир"', 500, 20, 'books');

-- Просмотр всех категорий ENUM-типа
SELECT enum_range(NULL::product_category);
```

### 2.4. Просмотр информации о типах

**Задание 2.7:** Получите информацию о созданных типах:

```sql
-- Все типы в текущей схеме
\dt

-- Информация о доменах
SELECT domain_name, data_type, is_nullable
FROM information_schema.domains
WHERE domain_schema = 'public';

-- Информация о составных типах
SELECT typname, typtype, typcategory 
FROM pg_type 
WHERE typtype IN ('c', 'e') AND typname NOT LIKE 'pg_%';
```

---

## Часть 3. Наследование таблиц (20 минут)

### 3.1. Создание иерархии наследования

**Задание 3.1:** Создайте иерархию таблиц для управления транспортными средствами:

```sql
-- Родительская таблица "vehicle"
CREATE TABLE vehicle (
    id SERIAL PRIMARY KEY,
    brand VARCHAR(50) NOT NULL,
    model VARCHAR(50) NOT NULL,
    year INTEGER CHECK (year >= 1900),
    price DECIMAL(12,2)
);

-- Дочерняя таблица "car" (наследует от vehicle)
CREATE TABLE car (
    body_type VARCHAR(30),
    doors_count INTEGER DEFAULT 4,
    transmission VARCHAR(20)
) INHERITS (vehicle);

-- Дочерняя таблица "motorcycle" (наследует от vehicle)
CREATE TABLE motorcycle (
    engine_cc INTEGER,
    type VARCHAR(30)
) INHERITS (vehicle);

-- Дочерняя таблица "truck" (наследует от vehicle)
CREATE TABLE truck (
    load_capacity_kg INTEGER,
    axle_count INTEGER DEFAULT 2
) INHERITS (vehicle);
```

**Задание 3.2:** Вставьте данные в дочерние таблицы:

```sql
INSERT INTO car (brand, model, year, price, body_type, doors_count, transmission)
VALUES ('Toyota', 'Camry', 2020, 2500000, 'sedan', 4, 'automatic');

INSERT INTO car (brand, model, year, price, body_type, doors_count, transmission)
VALUES ('BMW', 'X5', 2021, 5000000, 'SUV', 5, 'automatic');

INSERT INTO motorcycle (brand, model, year, price, engine_cc, type)
VALUES ('Harley-Davidson', 'Sportster', 2019, 1500000, 1200, 'cruiser');

INSERT INTO truck (brand, model, year, price, load_capacity_kg, axle_count)
VALUES ('KamAZ', '5490', 2020, 3000000, 15000, 3);
```

**Задание 3.3:** Выполните запросы к иерархии:

```sql
-- Запрос к родительской таблице (включает все дочерние)
SELECT * FROM vehicle WHERE price > 2000000;
-- Вернёт Toyota, BMW и KamAZ

-- Запрос ТОЛЬКО к родительской таблице (без дочерних)
SELECT * FROM ONLY vehicle;
-- Пустой результат (нет данных в родительской таблице)

-- Определение источника строки
SELECT 
    tableoid::regclass AS source_table,
    brand,
    model,
    price
FROM vehicle;
```

**Задание 3.4:** Найдите системный столбец `tableoid` и объясните его назначение.

```sql
-- Системный столбец tableoid показывает, из какой таблицы пришла строка
SELECT 
    tableoid::regclass AS source_table,
    COUNT(*) AS row_count
FROM vehicle
GROUP BY tableoid;
```

---

## Часть 4. Партиционирование с распределением по разным папкам (35 минут)

Для этого раздела мы создадим отдельные табличные пространства внутри каталога `/tmp/lab_postgres`. Все они будут физически располагаться в подкаталогах, созданных нами.

### 4.1. Подготовка табличных пространств для партиций

**Задание 4.1:** Создайте каталоги для разных секций внутри каталога лабораторной:

```bash
# В терминале (от root или с sudo)
sudo mkdir -p /tmp/lab_postgres/partitions/2024
sudo mkdir -p /tmp/lab_postgres/partitions/2025
sudo mkdir -p /tmp/lab_postgres/partitions/archive
sudo chown -R postgres:postgres /tmp/lab_postgres/partitions/
```

```sql
-- В psql (подключены к lab_db)
CREATE TABLESPACE ts_2024 LOCATION '/tmp/lab_postgres/partitions/2024';
CREATE TABLESPACE ts_2025 LOCATION '/tmp/lab_postgres/partitions/2025';
CREATE TABLESPACE ts_archive LOCATION '/tmp/lab_postgres/partitions/archive';
```

### 4.2. Создание секционированной таблицы

**Задание 4.2:** Создайте секционированную таблицу `orders_partitioned`:

```sql
-- Основная секционированная таблица
CREATE TABLE orders_partitioned (
    order_id SERIAL,
    customer_id INTEGER NOT NULL,
    order_date DATE NOT NULL,
    total_amount DECIMAL(12,2) NOT NULL,
    status VARCHAR(20) DEFAULT 'new'
) PARTITION BY RANGE (order_date);
```

### 4.3. Создание секций в разных табличных пространствах

**Задание 4.3:** Создайте секции, распределяя их по разным папкам:

```sql
-- Секция для 2024 года (на быстром диске)
CREATE TABLE orders_2024 PARTITION OF orders_partitioned
    FOR VALUES FROM ('2024-01-01') TO ('2025-01-01')
    TABLESPACE ts_2024;

-- Секция для 2025 года (на быстром диске)
CREATE TABLE orders_2025 PARTITION OF orders_partitioned
    FOR VALUES FROM ('2025-01-01') TO ('2026-01-01')
    TABLESPACE ts_2025;

-- Секция для архивных данных (на медленном диске)
CREATE TABLE orders_archive PARTITION OF orders_partitioned
    FOR VALUES FROM ('2020-01-01') TO ('2024-01-01')
    TABLESPACE ts_archive;

-- Секция по умолчанию (тоже на архивный диск)
CREATE TABLE orders_default PARTITION OF orders_partitioned
    DEFAULT TABLESPACE ts_archive;
```

### 4.4. Вставка и анализ данных

**Задание 4.4:** Вставьте данные, распределённые по разным годам:

```sql
INSERT INTO orders_partitioned (customer_id, order_date, total_amount) 
VALUES 
    (1, '2023-06-15', 1500.00),   -- попадает в orders_archive
    (2, '2024-03-20', 2200.00),   -- попадает в orders_2024
    (3, '2025-01-10', 1800.00),   -- попадает в orders_2025
    (4, '2025-07-05', 3100.00),   -- попадает в orders_2025
    (5, '2024-11-30', 950.00);    -- попадает в orders_2024
```

### 4.5. Анализ расположения данных

**Задание 4.5:** Проверьте, где физически хранятся данные:

```sql
-- 1. Проверка табличных пространств секций
SELECT 
    schemaname,
    tablename,
    tablespace
FROM pg_tables
WHERE tablename LIKE 'orders_%'
ORDER BY tablename;

-- 2. Проверка файловых путей для каждой секции
SELECT 
    'orders_2024' AS partition_name,
    pg_relation_filepath('orders_2024') AS file_path
UNION ALL
SELECT 
    'orders_2025',
    pg_relation_filepath('orders_2025')
UNION ALL
SELECT 
    'orders_archive',
    pg_relation_filepath('orders_archive');
```

**Задание 4.6:** Проверьте исключение секций (partition pruning) в планах запросов:

```sql
-- Запрос за 2024 год (должна сканироваться только секция 2024)
EXPLAIN (ANALYZE, BUFFERS)
SELECT * FROM orders_partitioned 
WHERE order_date BETWEEN '2024-01-01' AND '2024-12-31';

-- Запрос за все годы (сканируются все секции)
EXPLAIN (ANALYZE, BUFFERS)
SELECT * FROM orders_partitioned 
WHERE order_date > '2023-01-01';
```

### 4.6. Управление секциями

**Задание 4.7:** Добавьте новую секцию для 2026 года:

```bash
# В терминале создаём каталог
sudo mkdir -p /tmp/lab_postgres/partitions/2026
sudo chown postgres:postgres /tmp/lab_postgres/partitions/2026
```

```sql
-- В psql создаём табличное пространство
CREATE TABLESPACE ts_2026 LOCATION '/tmp/lab_postgres/partitions/2026';

-- Создаём секцию
CREATE TABLE orders_2026 PARTITION OF orders_partitioned
    FOR VALUES FROM ('2026-01-01') TO ('2027-01-01')
    TABLESPACE ts_2026;
```

**Задание 4.8:** Архивация старой секции (открепление и удаление):

```sql
-- Открепляем архивную секцию
ALTER TABLE orders_partitioned DETACH PARTITION orders_archive;

-- Удаляем таблицу (очень быстро, без VACUUM)
DROP TABLE orders_archive;

-- Если нужно, создаём новую архивную секцию на более медленном диске
CREATE TABLE orders_archive_new PARTITION OF orders_partitioned
    FOR VALUES FROM ('2023-01-01') TO ('2024-01-01')
    TABLESPACE ts_archive;
```

---

## Часть 5. Идентификаторы: SERIAL, IDENTITY, UUID, SEQUENCE (40 минут)

### 5.1. Сравнение SERIAL и IDENTITY

**Задание 5.1:** Создайте три версии одной таблицы с разными способами генерации ID:

```sql
-- Вариант 1: SERIAL (классический, НЕ стандарт SQL)
CREATE TABLE users_serial (
    id SERIAL PRIMARY KEY,
    name VARCHAR(100) NOT NULL,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);

-- Вариант 2: IDENTITY GENERATED BY DEFAULT (стандарт SQL)
CREATE TABLE users_identity_default (
    id INTEGER GENERATED BY DEFAULT AS IDENTITY PRIMARY KEY,
    name VARCHAR(100) NOT NULL,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);

-- Вариант 3: IDENTITY GENERATED ALWAYS (стандарт SQL)
CREATE TABLE users_identity_always (
    id INTEGER GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    name VARCHAR(100) NOT NULL,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);

-- Вариант 4: BIGSERIAL (для больших нагрузок)
CREATE TABLE users_bigserial (
    id BIGSERIAL PRIMARY KEY,
    name VARCHAR(100) NOT NULL,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);
```

**Задание 5.2:** Сравните поведение при вставке:

```sql
-- SERIAL: можно явно указать ID
INSERT INTO users_serial (id, name) VALUES (100, 'Иванов Иван');
-- Работает, но может нарушить уникальность

-- IDENTITY BY DEFAULT: можно явно указать ID
INSERT INTO users_identity_default (id, name) VALUES (100, 'Петров Пётр');
-- Работает, как и в SERIAL

-- IDENTITY ALWAYS: НЕЛЬЗЯ явно указать ID
INSERT INTO users_identity_always (id, name) VALUES (100, 'Сидоров Сидор');
-- ОШИБКА: cannot insert into column "id"
-- Правильно:
INSERT INTO users_identity_always (name) VALUES ('Сидоров Сидор');

-- Если очень нужно переопределить:
INSERT INTO users_identity_always (id, name) 
OVERRIDING SYSTEM VALUE 
VALUES (100, 'Сидоров Сидор');
-- Работает с OVERRIDING
```

**Задание 5.3:** Управление последовательностями:

```sql
-- Для SERIAL: нужно знать имя последовательности
SELECT pg_get_serial_sequence('users_serial', 'id');  -- 'users_serial_id_seq'
ALTER SEQUENCE users_serial_id_seq RESTART WITH 1000;

-- Для IDENTITY: не нужно знать имя последовательности
ALTER TABLE users_identity_default ALTER COLUMN id RESTART WITH 1000;
ALTER TABLE users_identity_always ALTER COLUMN id RESTART WITH 1000;
```

### 5.2. UUID

**Задание 5.4:** Создайте таблицу с UUID первичным ключом:

```sql
-- Включение расширения для UUID (если не установлено)
CREATE EXTENSION IF NOT EXISTS "uuid-ossp";

-- Таблица с UUID
CREATE TABLE orders_uuid (
    id UUID DEFAULT gen_random_uuid() PRIMARY KEY,
    customer_name VARCHAR(100) NOT NULL,
    order_date TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    total_amount DECIMAL(10,2)
);

-- Альтернативный вариант с явным указанием версии
CREATE TABLE orders_uuid_v1 (
    id UUID DEFAULT uuid_generate_v1() PRIMARY KEY,
    customer_name VARCHAR(100) NOT NULL,
    order_date TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    total_amount DECIMAL(10,2)
);

-- Вставка данных (ID генерируется автоматически)
INSERT INTO orders_uuid (customer_name, total_amount) 
VALUES ('Иванов Иван', 1500.00);

INSERT INTO orders_uuid (customer_name, total_amount) 
VALUES ('Петров Пётр', 2200.00);

-- Явная генерация UUID
SELECT gen_random_uuid();
SELECT uuid_generate_v1();
```

### 5.3. Sequence как отдельный объект

**Задание 5.5:** Создайте и используйте последовательности напрямую:

```sql
-- Создание последовательности
CREATE SEQUENCE invoice_number_seq
    INCREMENT BY 1
    MINVALUE 1
    MAXVALUE 999999
    START WITH 1000
    CACHE 10
    NO CYCLE;

-- Использование последовательности в таблице
CREATE TABLE invoices (
    invoice_id INTEGER PRIMARY KEY DEFAULT nextval('invoice_number_seq'),
    customer_name VARCHAR(100),
    invoice_date DATE DEFAULT CURRENT_DATE,
    amount DECIMAL(12,2)
);

-- Вставка с автоматической генерацией номера
INSERT INTO invoices (customer_name, amount) VALUES ('ООО "Ромашка"', 50000.00);
INSERT INTO invoices (customer_name, amount) VALUES ('ИП "Солнышко"', 30000.00);

-- Управление последовательностью
SELECT currval('invoice_number_seq');   -- текущее значение в сессии
SELECT nextval('invoice_number_seq');   -- следующее значение
SELECT setval('invoice_number_seq', 2000); -- установить значение
```

### 5.4. Сравнительный анализ идентификаторов

**Задание 5.6:** Создайте таблицу для сравнительного анализа:

```sql
-- Таблица с разными типами идентификаторов
CREATE TABLE id_comparison (
    id_serial SERIAL,
    id_identity INTEGER GENERATED BY DEFAULT AS IDENTITY,
    id_uuid UUID DEFAULT gen_random_uuid(),
    id_sequence INTEGER DEFAULT nextval('invoice_number_seq'),
    name VARCHAR(100)
);
```

**Задание 5.7:** Сравните размеры и производительность:

```sql
-- Размеры типов
SELECT 
    typname,
    typlen
FROM pg_type
WHERE typname IN ('int4', 'int8', 'uuid');

-- Результат: int4 = 4 байта, int8 = 8 байт, uuid = 16 байт
```

**Задание 5.8:** Заполните таблицу сравнительных характеристик:

```sql
-- Таблица для хранения результатов сравнения
CREATE TABLE id_analysis (
    method VARCHAR(20),
    type_name VARCHAR(20),
    size_bytes INTEGER,
    is_standard BOOLEAN,
    client_generation BOOLEAN,
    unique_global BOOLEAN,
    good_for_indexes BOOLEAN
);

-- Вставьте данные самостоятельно на основе изученного
```

## Часть 5. Финальное задание (20 минут)

### 5.1. Комплексное задание

**Задание 5.1:** Создайте полноценную схему для интернет-магазина, объединяющую все изученные темы:

```sql
-- 1. Домены
CREATE DOMAIN positive_money AS DECIMAL(12,2) CHECK (VALUE >= 0);
CREATE DOMAIN order_status AS VARCHAR(20) CHECK (VALUE IN ('new', 'paid', 'shipped', 'delivered', 'cancelled'));

-- 2. Составные типы
CREATE TYPE address AS (
    street TEXT,
    city VARCHAR(100),
    postal_code VARCHAR(20)
);

-- 3. Таблицы с IDENTITY
CREATE TABLE final_customers (
    id INTEGER GENERATED BY DEFAULT AS IDENTITY PRIMARY KEY,
    name TEXT NOT NULL,
    email TEXT UNIQUE,
    address address
);

CREATE TABLE final_products (
    id INTEGER GENERATED BY DEFAULT AS IDENTITY PRIMARY KEY,
    name TEXT NOT NULL,
    price positive_money NOT NULL,
    category TEXT
);

-- 4. Секционированная таблица с заказами (по датам) — используем уже созданные табличные пространства
CREATE TABLE final_orders (
    id INTEGER GENERATED BY DEFAULT AS IDENTITY,
    customer_id INTEGER NOT NULL REFERENCES final_customers(id),
    order_date DATE NOT NULL,
    status order_status DEFAULT 'new',
    total positive_money
) PARTITION BY RANGE (order_date);

-- 5. Секции
CREATE TABLE final_orders_2024 PARTITION OF final_orders
    FOR VALUES FROM ('2024-01-01') TO ('2025-01-01')
    TABLESPACE ts_2024;

CREATE TABLE final_orders_2025 PARTITION OF final_orders
    FOR VALUES FROM ('2025-01-01') TO ('2026-01-01')
    TABLESPACE ts_2025;

-- 6. Вставка данных
INSERT INTO final_customers (name, email, address) 
VALUES 
    ('Иванов Иван', 'ivan@mail.ru', ROW('ул. Ленина 1', 'Москва', '101000')::address),
    ('Петров Пётр', 'petr@mail.ru', ROW('ул. Пушкина 2', 'СПб', '192000')::address);

INSERT INTO final_orders (customer_id, order_date, total)
VALUES 
    (1, '2024-06-15', 1500.00),
    (2, '2025-01-20', 2200.00);
```

---

## Очистка окружения

После выполнения работы удалите созданные объекты и каталоги:

```sql
-- Переключитесь на другую базу (например, postgres)
\c postgres

-- Удаляем все таблицы и объекты в lab_db (каскадно)
DROP DATABASE IF EXISTS lab_db WITH (FORCE);
-- WITH (FORCE) принудительно завершает все подключения к БД

-- Удаляем табличные пространства (только после удаления БД)
DROP TABLESPACE IF EXISTS lab_tablespace;
DROP TABLESPACE IF EXISTS ts_2024;
DROP TABLESPACE IF EXISTS ts_2025;
DROP TABLESPACE IF EXISTS ts_archive;
DROP TABLESPACE IF EXISTS ts_2026;
```

```bash
# В терминале удаляем всю папку лабораторной
sudo rm -rf /tmp/lab_postgres
```
