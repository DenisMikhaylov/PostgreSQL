# Лабораторная работа: Стандартные типы данных в PostgreSQL



**Цель работы:** Изучить на практике стандартные типы данных PostgreSQL/Postgres Pro, научиться выбирать правильный тип для хранения данных, освоить операции с датой и временем, а также научиться преобразовывать и форматировать данные различных типов.


---

## Часть 0. Подготовка окружения (5 минут)

### 0.1. Создание базы данных для лабораторной

Выполните в терминале (под пользователем `postgres`):

```bash
sudo -u postgres psql
```

```sql
-- Создаём базу данных для лабораторной работы
CREATE DATABASE lab_types;

-- Подключаемся к ней
\c lab_types
```

---

## Часть 1. Числовые типы данных 

### 1.1. Обзор числовых типов

PostgreSQL поддерживает следующие числовые типы:

| Тип | Размер | Диапазон | Когда использовать |
|-----|--------|----------|-------------------|
| `SMALLINT` | 2 байта | -32 768 .. 32 767 | Маленькие числа (возраст, количество) |
| `INTEGER` | 4 байта | -2.1 млрд .. 2.1 млрд | Стандартные целые числа |
| `BIGINT` | 8 байт | -9.2·10¹⁸ .. 9.2·10¹⁸ | Большие числа (ID в крупных системах) |
| `DECIMAL(p,s)` / `NUMERIC(p,s)` | Переменный | Точность до 1000 цифр | Деньги, точные расчёты |
| `REAL` | 4 байта | ~6 десятичных знаков | Приближённые вычисления |
| `DOUBLE PRECISION` | 8 байт | ~15 десятичных знаков | Научные расчёты |

### 1.2. Создание таблицы с числовыми типами

**Задание 1.1:** Создайте таблицу для хранения информации о товарах:

```sql
CREATE TABLE products_numeric (
    product_id SERIAL PRIMARY KEY,
    name VARCHAR(100) NOT NULL,
    price DECIMAL(10,2) NOT NULL,          -- точная цена (2 знака после запятой)
    discount_percent DECIMAL(5,2),          -- скидка в процентах (до 99.99%)
    stock_quantity INTEGER DEFAULT 0,       -- количество на складе
    weight_kg REAL,                         -- вес с плавающей точкой
    rating DOUBLE PRECISION,                -- рейтинг с двойной точностью
    is_available SMALLINT DEFAULT 1         -- 1 - доступен, 0 - нет
);
```

**Задание 1.2:** Вставьте данные и выполните вычисления:

```sql
-- Вставка данных
INSERT INTO products_numeric (name, price, discount_percent, stock_quantity, weight_kg, rating)
VALUES 
    ('Ноутбук', 49999.99, 10.50, 15, 2.35, 4.8),
    ('Мышь', 999.99, 5.00, 50, 0.12, 4.5),
    ('Монитор', 15999.00, NULL, 8, 5.20, 4.2),
    ('Клавиатура', 2499.50, 0.00, 30, 0.85, 4.0);

-- Вычисление цены со скидкой
SELECT 
    name,
    price,
    discount_percent,
    price * (1 - COALESCE(discount_percent, 0) / 100) AS price_with_discount,
    stock_quantity,
    weight_kg
FROM products_numeric;
```

**Ожидаемый результат:** для ноутбука цена со скидкой = 49999.99 * (1 - 10.5/100) = 44749.99

### 1.3. Арифметические операции

**Задание 1.3:** Выполните различные арифметические операции:

```sql
-- Общая стоимость всех товаров на складе
SELECT 
    SUM(price * stock_quantity) AS total_inventory_value
FROM products_numeric;

-- Средняя цена товаров
SELECT 
    AVG(price) AS average_price,
    MIN(price) AS min_price,
    MAX(price) AS max_price
FROM products_numeric;

-- Округление чисел
SELECT 
    price,
    ROUND(price) AS rounded,
    ROUND(price, 1) AS rounded_1_decimal,
    CEIL(price) AS ceil_value,
    FLOOR(price) AS floor_value
FROM products_numeric;
```

---

## Часть 2. Строковые типы данных 

### 2.1. Обзор строковых типов

| Тип | Описание | Когда использовать |
|-----|----------|-------------------|
| `CHAR(n)` | Фиксированная длина, дополняется пробелами | Устаревший, не рекомендуется |
| `VARCHAR(n)` | Переменная длина с ограничением | Когда нужен лимит длины |
| `TEXT` | Переменная длина без ограничения | **Рекомендуется** для большинства случаев |

> **Важно:** В PostgreSQL нет значимой разницы в производительности между `VARCHAR` и `TEXT`. Используйте `TEXT` для гибкости.

### 2.2. Создание таблицы со строковыми типами

**Задание 2.1:** Создайте таблицу для хранения информации о пользователях:

```sql
CREATE TABLE users_string (
    user_id SERIAL PRIMARY KEY,
    username VARCHAR(50) UNIQUE NOT NULL,    -- ограничение длины
    email TEXT NOT NULL,                      -- неограниченная длина
    first_name VARCHAR(100),
    last_name VARCHAR(100),
    bio TEXT,                                 -- длинное описание
    status CHAR(1) DEFAULT 'A'                -- A - активен, I - неактивен
);
```

**Задание 2.2:** Вставьте и проанализируйте строковые данные:

```sql
-- Вставка данных
INSERT INTO users_string (username, email, first_name, last_name, bio, status)
VALUES 
    ('ivanov_i', 'ivan.ivanov@mail.ru', 'Иван', 'Иванов', 'Любит программировать', 'A'),
    ('petrov_p', 'petr@company.com', 'Пётр', 'Петров', NULL, 'A'),
    ('sidorov_s', 'sidorov@mail.ru', 'Сидор', 'Сидоров', 'Опытный разработчик с 10-летним стажем', 'I');

-- Строковые функции
SELECT 
    username,
    UPPER(username) AS username_upper,
    LENGTH(email) AS email_length,
    POSITION('@' IN email) AS at_position,
    SUBSTRING(email FROM POSITION('@' IN email) + 1) AS email_domain,
    COALESCE(bio, 'Нет описания') AS bio_display
FROM users_string;

-- Поиск по строке (регистронезависимый)
SELECT * FROM users_string 
WHERE email ILIKE '%mail.ru%';
```

### 2.3. Конкатенация и форматирование

**Задание 2.3:** Выполните конкатенацию строк:

```sql
-- Конкатенация (||)
SELECT 
    first_name || ' ' || last_name AS full_name,
    'Пользователь: ' || username || ' (' || email || ')' AS user_info
FROM users_string;

-- Функция CONCAT (обрабатывает NULL)
SELECT 
    CONCAT(first_name, ' ', last_name) AS full_name_concat,
    CONCAT_WS(' ', first_name, last_name) AS full_name_ws
FROM users_string;
```

---

## Часть 3. Логический тип BOOLEAN 

### 3.1. Создание и использование BOOLEAN

**Задание 3.1:** Создайте таблицу с булевыми полями:

```sql
CREATE TABLE tasks_boolean (
    task_id SERIAL PRIMARY KEY,
    title TEXT NOT NULL,
    is_completed BOOLEAN DEFAULT FALSE,
    is_urgent BOOLEAN,
    is_archived BOOLEAN DEFAULT FALSE
);

-- Вставка данных (разные способы указания TRUE/FALSE)
INSERT INTO tasks_boolean (title, is_completed, is_urgent, is_archived)
VALUES 
    ('Купить продукты', true, false, false),
    ('Сдать отчёт', FALSE, TRUE, FALSE),
    ('Написать письмо', 'yes', 'no', 'off'),
    ('Позвонить клиенту', 1, 0, 0);

-- Запросы с BOOLEAN
SELECT 
    title,
    is_completed,
    is_urgent,
    CASE WHEN is_completed THEN 'Выполнено' ELSE 'Не выполнено' END AS status_text
FROM tasks_boolean;

-- Фильтрация по булевым значениям
SELECT * FROM tasks_boolean WHERE is_completed = true;
SELECT * FROM tasks_boolean WHERE is_urgent IS NOT FALSE;
```

---

## Часть 4. Типы даты и времени

Это наиболее важная часть лабораторной работы. Типы даты и времени — одни из самых часто используемых в реальных приложениях.

### 4.1. Обзор типов даты и времени

PostgreSQL поддерживает следующие типы для работы с датой и временем:

| Тип | Размер | Описание | Пример |
|-----|--------|----------|--------|
| `DATE` | 4 байта | Только дата (без времени) | `2024-01-15` |
| `TIME` | 8 байт | Только время (без даты) | `14:30:00` |
| `TIMESTAMP` | 8 байт | Дата и время (без часового пояса) | `2024-01-15 14:30:00` |
| `TIMESTAMPTZ` | 8 байт | Дата и время **с часовым поясом** | `2024-01-15 14:30:00+03` |
| `INTERVAL` | 16 байт | Временной интервал | `1 day 2 hours` |

**Ключевые особенности:**
- Точность всех типов — **1 микросекунда**
- `TIMESTAMP` по стандарту SQL означает `TIMESTAMP WITHOUT TIME ZONE`
- `TIMESTAMPTZ` — это расширение PostgreSQL (сокращение от `TIMESTAMP WITH TIME ZONE`)
- Можно задавать точность `p` (0–6) для секунд: `TIMESTAMP(3)` хранит миллисекунды

### 4.2. Создание таблицы с типами даты и времени

**Задание 4.1:** Создайте таблицу для хранения событий:

```sql
CREATE TABLE events_datetime (
    event_id SERIAL PRIMARY KEY,
    event_name TEXT NOT NULL,
    event_date DATE NOT NULL,                  -- только дата
    start_time TIME,                           -- только время
    start_timestamp TIMESTAMP,                 -- дата+время без часового пояса
    start_timestamptz TIMESTAMPTZ,             -- дата+время с часовым поясом
    duration INTERVAL,                         -- продолжительность
    created_at TIMESTAMPTZ DEFAULT NOW()       -- автоматическая метка времени
);
```

**Задание 4.2:** Вставьте данные с различными форматами:

```sql
-- Различные способы указания даты и времени
INSERT INTO events_datetime (
    event_name, 
    event_date, 
    start_time, 
    start_timestamp, 
    start_timestamptz, 
    duration
)
VALUES 
    (
        'Конференция по PostgreSQL',
        '2024-05-15',                    -- DATE: ISO формат
        '09:00:00',                      -- TIME: HH:MM:SS
        '2024-05-15 09:00:00',           -- TIMESTAMP
        '2024-05-15 09:00:00+03',        -- TIMESTAMPTZ (Москва)
        '2 hours 30 minutes'             -- INTERVAL
    ),
    (
        'Вебинар',
        '2024/05/20',                    -- DATE: альтернативный формат
        '15:30',                         -- TIME: без секунд
        '2024-05-20 15:30:00',
        '2024-05-20 12:30:00Z',          -- TIMESTAMPTZ (UTC)
        '1 hour'
    ),
    (
        'Хакатон',
        'May 25, 2024',                  -- DATE: текстовый формат
        '10:00:00',
        '2024-05-25 10:00:00',
        '2024-05-25 10:00:00-04',        -- TIMESTAMPTZ (Нью-Йорк)
        '3 days'                         -- INTERVAL
    );
```

### 4.3. Системные функции для получения текущего времени

**Задание 4.3:** Изучите функции для получения текущей даты и времени:

```sql
-- Текущая дата и время
SELECT 
    CURRENT_DATE AS current_date,                     -- только дата
    CURRENT_TIME AS current_time,                     -- только время (с часовым поясом)
    CURRENT_TIMESTAMP AS current_timestamp,           -- дата+время (с часовым поясом)
    LOCALTIME AS localtime,                           -- время (без часового пояса)
    LOCALTIMESTAMP AS localtimestamp;                 -- дата+время (без часового пояса)

-- Специальные литералы
SELECT 
    'now'::TIMESTAMPTZ AS now_literal,
    'epoch'::TIMESTAMPTZ AS epoch,                    -- 1970-01-01 00:00:00 UTC
    'infinity'::TIMESTAMPTZ AS infinity,              -- бесконечность
    '-infinity'::TIMESTAMPTZ AS minus_infinity;       -- минус бесконечность
```

### 4.4. Арифметика с датами и временем

**Задание 4.4:** Выполните арифметические операции с датами:

```sql
-- Сложение и вычитание с датами
SELECT 
    event_name,
    event_date,
    event_date + 7 AS next_week,                     -- DATE + INTEGER → DATE
    event_date + INTERVAL '1 month' AS next_month,   -- DATE + INTERVAL → TIMESTAMP
    event_date - INTERVAL '1 year' AS last_year;     -- DATE - INTERVAL → TIMESTAMP

-- Вычитание дат
SELECT 
    event_name,
    event_date,
    CURRENT_DATE - event_date AS days_ago,           -- DATE - DATE → INTEGER
    AGE(CURRENT_DATE, event_date) AS age             -- AGE → INTERVAL
FROM events_datetime;

-- Сложение времени и интервалов
SELECT 
    event_name,
    start_time,
    start_time + INTERVAL '30 minutes' AS plus_30min,
    start_time + duration AS calculated_end_time     -- TIME + INTERVAL → TIME
FROM events_datetime;

-- Сложение TIMESTAMP и INTERVAL
SELECT 
    event_name,
    start_timestamp,
    start_timestamp + duration AS end_timestamp,
    start_timestamp + INTERVAL '2 hours' AS plus_2h
FROM events_datetime;
```

### 4.5. Функции EXTRACT и DATE_PART

**Задание 4.5:** Извлеките отдельные части даты:

```sql
-- EXTRACT — извлечение частей даты
SELECT 
    event_name,
    start_timestamptz,
    EXTRACT(YEAR FROM start_timestamptz) AS year,
    EXTRACT(MONTH FROM start_timestamptz) AS month,
    EXTRACT(DAY FROM start_timestamptz) AS day,
    EXTRACT(HOUR FROM start_timestamptz) AS hour,
    EXTRACT(MINUTE FROM start_timestamptz) AS minute,
    EXTRACT(DOW FROM start_timestamptz) AS day_of_week,       -- 0=воскресенье
    EXTRACT(DOY FROM start_timestamptz) AS day_of_year,
    EXTRACT(WEEK FROM start_timestamptz) AS week_number,
    EXTRACT(QUARTER FROM start_timestamptz) AS quarter,
    EXTRACT(EPOCH FROM start_timestamptz) AS epoch_seconds   -- секунды с 1970-01-01
FROM events_datetime;

-- DATE_PART — альтернатива EXTRACT
SELECT 
    event_name,
    DATE_PART('year', start_timestamptz) AS year,
    DATE_PART('month', start_timestamptz) AS month
FROM events_datetime;
```

### 4.6. Функция DATE_TRUNC

**Задание 4.6:** Округлите даты до нужной точности:

```sql
-- DATE_TRUNC — усечение даты до указанной точности
SELECT 
    start_timestamptz,
    DATE_TRUNC('year', start_timestamptz) AS start_of_year,
    DATE_TRUNC('month', start_timestamptz) AS start_of_month,
    DATE_TRUNC('day', start_timestamptz) AS start_of_day,
    DATE_TRUNC('hour', start_timestamptz) AS start_of_hour,
    DATE_TRUNC('week', start_timestamptz) AS start_of_week
FROM events_datetime;
```

### 4.7. Форматирование дат (TO_CHAR)

**Задание 4.7:** Отформатируйте даты в нужном виде:

```sql
-- TO_CHAR — форматирование даты в строку
SELECT 
    start_timestamptz,
    TO_CHAR(start_timestamptz, 'YYYY-MM-DD') AS iso_date,
    TO_CHAR(start_timestamptz, 'DD.MM.YYYY') AS ru_date,
    TO_CHAR(start_timestamptz, 'HH24:MI:SS') AS time_24h,
    TO_CHAR(start_timestamptz, 'HH12:MI AM') AS time_12h,
    TO_CHAR(start_timestamptz, 'Month DD, YYYY') AS full_date,
    TO_CHAR(start_timestamptz, 'Day') AS day_name,
    TO_CHAR(start_timestamptz, 'DDD') AS day_of_year
FROM events_datetime;
```

**Популярные форматы TO_CHAR:**

| Шаблон | Описание | Пример |
|--------|----------|--------|
| `YYYY` | Год (4 цифры) | 2024 |
| `YY` | Год (2 цифры) | 24 |
| `MM` | Месяц (2 цифры) | 05 |
| `MON` | Сокращённое название месяца | May |
| `MONTH` | Полное название месяца | May |
| `DD` | День месяца (2 цифры) | 15 |
| `Day` | День недели | Monday |
| `HH24` | Час (24-часовой) | 14 |
| `HH12` | Час (12-часовой) | 02 |
| `MI` | Минуты | 30 |
| `SS` | Секунды | 00 |

### 4.8. Тип INTERVAL — подробно

**Задание 4.8:** Изучите работу с интервалами:

```sql
-- Создание интервалов
SELECT 
    INTERVAL '1 year 2 months 3 days' AS interval1,
    INTERVAL '2 hours 30 minutes' AS interval2,
    INTERVAL '1.5 days' AS interval3,
    INTERVAL 'P1Y2M3DT4H5M6S' AS iso_interval;        -- ISO 8601 формат

-- Извлечение частей интервала
SELECT 
    duration,
    EXTRACT(DAY FROM duration) AS days,
    EXTRACT(HOUR FROM duration) AS hours,
    EXTRACT(MINUTE FROM duration) AS minutes
FROM events_datetime;

-- Суммирование интервалов
SELECT 
    SUM(duration) AS total_duration,
    AVG(duration) AS avg_duration
FROM events_datetime;

-- JUSTIFY — нормализация интервалов
SELECT 
    duration,
    JUSTIFY_DAYS(duration) AS justified_days,
    JUSTIFY_HOURS(duration) AS justified_hours,
    JUSTIFY_INTERVAL(duration) AS justified_interval
FROM events_datetime;
```

### 4.9. Часовые пояса и TIMESTAMPTZ

**Задание 4.9:** Изучите работу с часовыми поясами:

```sql
-- Просмотр текущего часового пояса
SHOW TIMEZONE;

-- Изменение часового пояса для сессии
SET TIMEZONE = 'Europe/Moscow';
-- SET TIMEZONE = 'UTC';
-- SET TIMEZONE = 'America/New_York';

-- Сравнение TIMESTAMP и TIMESTAMPTZ
SELECT 
    start_timestamp AS without_tz,
    start_timestamptz AS with_tz,
    start_timestamp AT TIME ZONE 'UTC' AS timestamp_as_utc,
    start_timestamptz AT TIME ZONE 'Europe/Moscow' AS timestamptz_in_msk
FROM events_datetime;

-- Список доступных часовых поясов
SELECT name FROM pg_timezone_names ORDER BY name LIMIT 20;
```

---

## Часть 5. Диапазонные типы (15 минут)

### 5.1. Обзор диапазонных типов

PostgreSQL поддерживает диапазонные типы для различных данных:

| Тип | Описание |
|-----|----------|
| `int4range` | Диапазон целых чисел |
| `int8range` | Диапазон больших целых |
| `numrange` | Диапазон чисел с плавающей точкой |
| `tsrange` | Диапазон временных меток (без часового пояса) |
| `tstzrange` | Диапазон временных меток (с часовым поясом) |
| `daterange` | Диапазон дат |

### 5.2. Создание и использование диапазонов

**Задание 5.1:** Создайте таблицу с диапазонами дат:

```sql
CREATE TABLE reservations_range (
    reservation_id SERIAL PRIMARY KEY,
    room_number INTEGER NOT NULL,
    guest_name TEXT NOT NULL,
    stay_period daterange NOT NULL,              -- диапазон дат
    price_range numrange,                        -- диапазон цен
    check_in_time tstzrange                      -- диапазон времени заезда
);

-- Вставка данных с диапазонами
INSERT INTO reservations_range (room_number, guest_name, stay_period, price_range, check_in_time)
VALUES 
    (
        101, 
        'Иванов Иван', 
        '[2024-06-01, 2024-06-05)',              -- с 1 по 4 июня (5-го выезд)
        '[5000, 7000]',                          -- цена от 5000 до 7000
        '[2024-06-01 14:00:00+03, 2024-06-01 20:00:00+03]'
    ),
    (
        102,
        'Петров Пётр',
        '[2024-06-10, 2024-06-15)',
        '[8000, 10000]',
        '[2024-06-10 15:00:00+03, 2024-06-10 22:00:00+03]'
    );

-- Запросы с диапазонами
SELECT 
    guest_name,
    LOWER(stay_period) AS check_in,
    UPPER(stay_period) AS check_out,
    UPPER(stay_period) - LOWER(stay_period) AS nights,
    LOWER(price_range) AS min_price,
    UPPER(price_range) AS max_price
FROM reservations_range;

-- Поиск пересечений (пересекающиеся бронирования)
SELECT 
    r1.guest_name AS guest1,
    r2.guest_name AS guest2,
    r1.room_number
FROM reservations_range r1
JOIN reservations_range r2 
    ON r1.room_number = r2.room_number 
    AND r1.reservation_id < r2.reservation_id
    AND r1.stay_period && r2.stay_period;        -- && — оператор пересечения
```

---



## 🧹 Очистка окружения

После выполнения работы удалите созданные объекты:

```sql
-- Удаление всех таблиц
DROP TABLE IF EXISTS products_numeric CASCADE;
DROP TABLE IF EXISTS users_string CASCADE;
DROP TABLE IF EXISTS tasks_boolean CASCADE;
DROP TABLE IF EXISTS events_datetime CASCADE;
DROP TABLE IF EXISTS reservations_range CASCADE;
DROP TABLE IF EXISTS products_json CASCADE;
DROP TABLE IF EXISTS orders_complete CASCADE;

-- Переключение на другую базу
\c postgres

-- Удаление базы данных
DROP DATABASE IF EXISTS lab_types;
```

---

**Успешной работы!**
