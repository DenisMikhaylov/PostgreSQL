# Лабораторная работа: Мониторинг активности и настройка логирования


**Цель работы:** Изучить на практике настройку логирования PostgreSQL/Postgres Pro (log_destination, logging_collector, уровни логирования, логирование медленных запросов) и освоить инструменты мониторинга активности: `pg_stat_activity` и `pg_stat_statements`.


## Часть 0. Подготовка окружения 

### 0.1. Создание базы данных и пользователя

```sql
-- Создаём базу данных для лабораторной работы
CREATE DATABASE lab_monitoring;

-- Подключаемся к ней
\c lab_monitoring

-- Создаём тестового пользователя (для демонстрации разных сессий)
CREATE USER test_user WITH PASSWORD 'test123';
GRANT ALL PRIVILEGES ON DATABASE lab_monitoring TO test_user;
```

### 0.2. Создание тестовых объектов

```sql
-- Создаём таблицу для тестовых запросов
CREATE TABLE test_data (
    id SERIAL PRIMARY KEY,
    name TEXT NOT NULL,
    value INTEGER,
    created_at TIMESTAMP DEFAULT NOW()
);

-- Заполняем тестовыми данными (100 000 строк)
INSERT INTO test_data (name, value)
SELECT 
    'Item_' || generate_series,
    floor(random() * 1000)::int
FROM generate_series(1, 100000);

-- Создаём индексы
CREATE INDEX idx_test_data_value ON test_data(value);
```


## Часть 1. Настройка логирования 

### 1.1. Просмотр текущих настроек логирования

**Задание 1.1:** Посмотрите текущие настройки логирования:

```sql
-- Основные параметры логирования
SELECT name, setting, unit, context
FROM pg_settings
WHERE name IN (
    'log_destination',
    'logging_collector',
    'log_directory',
    'log_filename',
    'log_min_messages',
    'log_min_error_statement',
    'log_min_duration_statement',
    'log_statement',
    'log_line_prefix'
)
ORDER BY name;
```

**Задание 1.2:** Запишите текущие значения в таблицу:

```sql
CREATE TABLE lab_log_settings_backup AS
SELECT name, setting, unit, context
FROM pg_settings
WHERE name IN (
    'log_destination',
    'logging_collector',
    'log_directory',
    'log_filename',
    'log_min_messages',
    'log_min_error_statement',
    'log_min_duration_statement',
    'log_statement',
    'log_line_prefix'
);
```

### 1.2. Настройка log_destination и logging_collector

Для изменения параметров логирования в PostgreSQL 17+ можно использовать команду `ALTER SYSTEM`:

```sql
-- Включаем сборщик логов
ALTER SYSTEM SET logging_collector = 'on';

-- Настраиваем вывод в CSV и stderr
ALTER SYSTEM SET log_destination = 'stderr, csvlog';

-- Настраиваем каталог для логов
ALTER SYSTEM SET log_directory = 'pg_log';

-- Настраиваем шаблон имени файла
ALTER SYSTEM SET log_filename = 'postgresql-%Y-%m-%d_%H%M%S.log';

-- Права доступа к файлам логов
ALTER SYSTEM SET log_file_mode = '0640';

-- Ротация по времени (1 день)
ALTER SYSTEM SET log_rotation_age = '1d';

-- Ротация по размеру (100 МБ)
ALTER SYSTEM SET log_rotation_size = '100MB';

-- Усечение при ротации
ALTER SYSTEM SET log_truncate_on_rotation = 'on';
```

**Задание 1.3:** Примените изменения:

```sql
-- Перезагрузка конфигурации
SELECT pg_reload_conf();

-- Проверка новых настроек
SELECT name, setting 
FROM pg_settings 
WHERE name IN ('logging_collector', 'log_destination', 'log_directory');
```

### 1.3. Настройка уровней логирования

**Задание 1.4:** Настройте уровни логирования:

```sql
-- Логируем сообщения уровня WARNING и выше
ALTER SYSTEM SET log_min_messages = 'warning';

-- Логируем операторы, завершившиеся с ERROR и выше
ALTER SYSTEM SET log_min_error_statement = 'error';

-- Логируем запросы, выполняющиеся более 1 секунды
ALTER SYSTEM SET log_min_duration_statement = '1000';

-- Логируем DDL и DML команды
ALTER SYSTEM SET log_statement = 'mod';

-- Настраиваем формат строки логирования
ALTER SYSTEM SET log_line_prefix = '%m [%p] %u@%d %r %a ';

-- Дополнительные параметры
ALTER SYSTEM SET log_connections = 'on';
ALTER SYSTEM SET log_disconnections = 'on';
ALTER SYSTEM SET log_checkpoints = 'on';
ALTER SYSTEM SET log_lock_waits = 'on';

-- Применяем
SELECT pg_reload_conf();
```

### 1.4. Тестирование логирования

**Задание 1.5:** Создайте события, которые должны попасть в лог:

```sql
-- 1. Медленный запрос (более 1 секунды)
-- Создаём функцию для генерации медленного запроса
CREATE OR REPLACE FUNCTION slow_query()
RETURNS INTEGER AS $$
DECLARE
    v_result INTEGER;
BEGIN
    -- Выполняем медленный запрос
    SELECT count(*) INTO v_result
    FROM test_data t1
    CROSS JOIN test_data t2
    WHERE t1.value = t2.value;
    RETURN v_result;
END;
$$ LANGUAGE plpgsql;

-- Вызываем медленный запрос
SELECT slow_query();

-- 2. DDL операция (CREATE)
CREATE TABLE log_test_table (
    id SERIAL PRIMARY KEY,
    name TEXT
);

-- 3. DML операция (INSERT, UPDATE, DELETE)
INSERT INTO log_test_table (name) VALUES ('test1'), ('test2'), ('test3');
UPDATE log_test_table SET name = 'updated' WHERE id = 1;
DELETE FROM log_test_table WHERE id = 2;

-- 4. Операция с ошибкой
-- Попытка вставить дубликат в PRIMARY KEY (вызовет ошибку)
INSERT INTO log_test_table (id, name) VALUES (1, 'duplicate');
```

**Задание 1.6:** Проверьте содержимое логов:

```sql
-- Где хранятся логи?
SELECT setting FROM pg_settings WHERE name = 'log_directory';
SELECT setting FROM pg_settings WHERE name = 'data_directory';

-- Функция для чтения логов (если настроен CSV)
-- Создаём представление для просмотра CSV-логов
CREATE EXTENSION IF NOT EXISTS file_fdw;

CREATE SERVER pg_log_server FOREIGN DATA WRAPPER file_fdw;

CREATE FOREIGN TABLE pg_log_csv (
    log_time TIMESTAMP(3) WITH TIME ZONE,
    user_name TEXT,
    database_name TEXT,
    process_id INTEGER,
    connection_from TEXT,
    session_id TEXT,
    session_line_num BIGINT,
    command_tag TEXT,
    session_start_time TIMESTAMP WITH TIME ZONE,
    virtual_transaction_id TEXT,
    transaction_id BIGINT,
    error_severity TEXT,
    sql_state_code TEXT,
    message TEXT,
    detail TEXT,
    hint TEXT,
    internal_query TEXT,
    internal_query_pos INTEGER,
    context TEXT,
    query TEXT,
    query_pos INTEGER,
    location TEXT,
    application_name TEXT
)
SERVER pg_log_server
OPTIONS (
    filename '/var/lib/postgresql/17/main/pg_log/postgresql.csv',
    format 'csv',
    header 'true'
);

```

## Часть 2. Мониторинг активности: pg_stat_activity 

### 2.1. Просмотр текущих подключений

**Задание 2.1:** Изучите текущие подключения:

```sql
-- Просмотр всех активных сессий
SELECT 
    pid,
    usename,
    datname,
    application_name,
    client_addr,
    state,
    backend_start,
    xact_start,
    query_start,
    state_change,
    left(query, 80) AS query_preview
FROM pg_stat_activity
ORDER BY backend_start;

-- Количество подключений по базам данных
SELECT 
    datname,
    count(*) AS connections,
    count(*) FILTER (WHERE state = 'active') AS active,
    count(*) FILTER (WHERE state = 'idle in transaction') AS idle_in_transaction
FROM pg_stat_activity
GROUP BY datname
ORDER BY connections DESC;
```

### 2.2. Анализ состояний сессий

**Задание 2.2:** Создайте сессии в разных состояниях для наблюдения:

**В ОТДЕЛЬНОМ ТЕРМИНАЛЕ 1:**

```bash
# Подключение 1 - создание долгой транзакции
sudo -u postgres psql -d lab_monitoring
```

```sql
-- Начинаем транзакцию, но не завершаем
BEGIN;
SELECT count(*) FROM test_data;
-- Сессия теперь в состоянии "idle in transaction"
```

**В ОТДЕЛЬНОМ ТЕРМИНАЛЕ 2:**

```bash
# Подключение 2 - долгий запрос
sudo -u postgres psql -d lab_monitoring
```

```sql
-- Долгий запрос (используем sleep для имитации)
SELECT pg_sleep(60);
-- Или медленный запрос
SELECT count(*) FROM test_data t1 CROSS JOIN test_data t2 
WHERE t1.value = t2.value;
```

**В ОТДЕЛЬНОМ ТЕРМИНАЛЕ 3:**

```bash
# Подключение 3 - обычное подключение
sudo -u postgres psql -d lab_monitoring
```

```sql
-- Обычный запрос
SELECT * FROM test_data LIMIT 5;
```

**Задание 2.3:** В основном подключении выполните запросы для анализа:

```sql
-- Поиск сессий в состоянии "idle in transaction"
SELECT 
    pid,
    usename,
    datname,
    client_addr,
    now() - xact_start AS transaction_duration,
    now() - query_start AS query_duration,
    query
FROM pg_stat_activity
WHERE state = 'idle in transaction'
  AND xact_start IS NOT NULL
ORDER BY transaction_duration DESC;

-- Поиск активных долгих запросов
SELECT 
    pid,
    usename,
    datname,
    state,
    now() - query_start AS duration_seconds,
    left(query, 150) AS query
FROM pg_stat_activity
WHERE state = 'active'
  AND now() - query_start > interval '1 second'
ORDER BY duration_seconds DESC;

-- Поиск сессий с определённым application_name
SELECT 
    pid,
    usename,
    application_name,
    state,
    now() - query_start AS duration_seconds,
    query
FROM pg_stat_activity
WHERE application_name = 'psql'
  AND usename = 'postgres';
```

### 2.3. Выявление блокировок

**Задание 2.4:** Создайте ситуацию блокировки для исследования:

**В ОТДЕЛЬНОМ ТЕРМИНАЛЕ 1:**

```sql
-- Начинаем транзакцию и блокируем таблицу
BEGIN;
LOCK TABLE test_data IN ACCESS EXCLUSIVE MODE;
-- Держим блокировку, не завершая транзакцию
-- (оставляем сессию открытой)
```

**В ОТДЕЛЬНОМ ТЕРМИНАЛЕ 2:**

```sql
-- Пытаемся выполнить запрос к заблокированной таблице
SELECT count(*) FROM test_data;
-- Этот запрос будет ожидать снятия блокировки
```

**В ОСНОВНОМ ТЕРМИНАЛЕ:**

```sql
-- Поиск блокировок
SELECT 
    blocked.pid AS blocked_pid,
    blocked.usename AS blocked_user,
    blocked.query AS blocked_query,
    blocking.pid AS blocking_pid,
    blocking.usename AS blocking_user,
    blocking.query AS blocking_query,
    blocked.wait_event_type,
    blocked.wait_event
FROM pg_stat_activity AS blocked
JOIN pg_stat_activity AS blocking 
    ON blocking.pid = ANY(pg_blocking_pids(blocked.pid))
WHERE blocked.pid <> pg_backend_pid();

-- Детальная информация о блокировках
SELECT 
    locktype,
    relation::regclass AS relation,
    pid,
    mode,
    granted
FROM pg_locks
WHERE NOT granted
ORDER BY pid;
```

### 2.4. Завершение проблемных сессий

**Задание 2.5:** Выполните управление сессиями:

```sql
-- Отмена текущего запроса в сессии (без завершения сессии)
-- Найти PID проблемной сессии и отменить запрос
SELECT pg_cancel_backend(<PID>);

-- Полное завершение сессии
SELECT pg_terminate_backend(<PID>);

-- Завершение всех "idle in transaction" сессий старше 5 минут
SELECT pg_terminate_backend(pid)
FROM pg_stat_activity
WHERE state = 'idle in transaction'
  AND now() - xact_start > interval '5 minutes'
  AND pid <> pg_backend_pid();
```

### 2.5. Мониторинг системных ресурсов

**Задание 2.6:** Создайте функцию для мониторинга активности:

```sql
-- Создаём представление для мониторинга сессий
CREATE VIEW session_monitor AS
SELECT 
    now() AS check_time,
    count(*) AS total_sessions,
    count(*) FILTER (WHERE state = 'active') AS active_sessions,
    count(*) FILTER (WHERE state = 'idle in transaction') AS idle_transaction_sessions,
    count(*) FILTER (WHERE state = 'idle') AS idle_sessions,
    round(100.0 * count(*) FILTER (WHERE state = 'active') / count(*), 2) AS active_percent,
    round(100.0 * count(*) FILTER (WHERE state = 'idle in transaction') / count(*), 2) AS idle_transaction_percent
FROM pg_stat_activity;

-- Просмотр метрик
SELECT * FROM session_monitor;
```


## Часть 3. Мониторинг производительности: pg_stat_statements 

### 3.1. Установка pg_stat_statements

Если расширение ещё не установлено, выполните:

```sql
-- Проверка, установлено ли расширение
SELECT * FROM pg_extension WHERE extname = 'pg_stat_statements';

-- Если не установлено:
CREATE EXTENSION IF NOT EXISTS pg_stat_statements;

-- Проверка параметров pg_stat_statements
SELECT 
    name, 
    setting, 
    unit,
    short_desc
FROM pg_settings
WHERE name LIKE 'pg_stat_statements%';
```

### 3.2. Сбор статистики запросов

**Задание 3.1:** Выполните различные запросы для накопления статистики:

```sql
-- 1. Простые SELECT-запросы
SELECT * FROM test_data WHERE value BETWEEN 100 AND 200 LIMIT 10;
SELECT * FROM test_data WHERE value BETWEEN 200 AND 300 LIMIT 10;
SELECT * FROM test_data WHERE value BETWEEN 300 AND 400 LIMIT 10;

-- 2. Агрегирующие запросы
SELECT value, count(*) FROM test_data GROUP BY value ORDER BY value LIMIT 10;
SELECT avg(value) FROM test_data;
SELECT max(value), min(value) FROM test_data;

-- 3. Запросы с JOIN (самособъединение)
SELECT t1.value, count(*)
FROM test_data t1
JOIN test_data t2 ON t1.value = t2.value
GROUP BY t1.value
LIMIT 5;

-- 4. Запросы с ORDER BY
SELECT * FROM test_data ORDER BY value DESC LIMIT 100;

-- 5. Обновление данных
UPDATE test_data SET value = value + 1 WHERE id % 10 = 0;

-- 6. Вставка данных
INSERT INTO test_data (name, value) 
SELECT 'New_' || generate_series, floor(random() * 1000)::int
FROM generate_series(1, 100);

-- 7. Удаление данных
DELETE FROM test_data WHERE id % 100 = 0;
```

### 3.3. Анализ статистики pg_stat_statements

**Задание 3.2:** Выполните аналитические запросы:

```sql
-- 1. Топ-10 запросов по общему времени выполнения
SELECT 
    queryid,
    calls,
    total_time / 1000 AS total_seconds,
    mean_time AS avg_ms,
    max_time AS max_ms,
    rows,
    left(query, 150) AS query_preview
FROM pg_stat_statements
ORDER BY total_time DESC
LIMIT 10;

-- 2. Топ-10 запросов по количеству вызовов
SELECT 
    queryid,
    calls,
    total_time / 1000 AS total_seconds,
    mean_time AS avg_ms,
    rows,
    left(query, 150) AS query_preview
FROM pg_stat_statements
ORDER BY calls DESC
LIMIT 10;

-- 3. Топ-10 запросов по среднему времени выполнения
SELECT 
    queryid,
    calls,
    mean_time AS avg_ms,
    total_time / 1000 AS total_seconds,
    rows,
    left(query, 150) AS query_preview
FROM pg_stat_statements
WHERE calls > 10
ORDER BY mean_time DESC
LIMIT 10;

-- 4. Запросы с большим количеством временных блоков
SELECT 
    queryid,
    calls,
    temp_blks_read,
    temp_blks_written,
    temp_blks_read + temp_blks_written AS total_temp_blocks,
    left(query, 150) AS query_preview
FROM pg_stat_statements
WHERE temp_blks_read + temp_blks_written > 1000
ORDER BY total_temp_blocks DESC
LIMIT 10;

-- 5. Запросы с низким коэффициентом попадания в кэш
SELECT 
    queryid,
    calls,
    shared_blks_hit,
    shared_blks_read,
    CASE 
        WHEN shared_blks_hit + shared_blks_read > 0 
        THEN round(100.0 * shared_blks_hit / (shared_blks_hit + shared_blks_read), 2)
        ELSE 0
    END AS cache_hit_ratio,
    left(query, 150) AS query_preview
FROM pg_stat_statements
WHERE shared_blks_hit + shared_blks_read > 1000
ORDER BY cache_hit_ratio ASC
LIMIT 10;
```

### 3.4. Анализ по пользователям и базам данных

**Задание 3.3:** Выполните анализ по пользователям:

```sql
-- Статистика по пользователям
SELECT 
    pg_roles.rolname AS username,
    pg_database.datname AS database,
    count(*) AS query_count,
    sum(total_time) / 1000 AS total_seconds,
    sum(calls) AS total_calls,
    round(avg(mean_time), 2) AS avg_time_ms
FROM pg_stat_statements
JOIN pg_roles ON pg_stat_statements.userid = pg_roles.oid
JOIN pg_database ON pg_stat_statements.dbid = pg_database.oid
GROUP BY pg_roles.rolname, pg_database.datname
ORDER BY total_seconds DESC;
```

### 3.5. Сравнение планов запросов

**Задание 3.4:** Проанализируйте план конкретного запроса:

```sql
-- Находим queryid самого медленного запроса
SELECT queryid, query, total_time
FROM pg_stat_statements
ORDER BY total_time DESC
LIMIT 1;

-- Анализируем план запроса (подставьте queryid)
-- PostgreSQL 14+ поддерживает pg_stat_statements_info
SELECT 
    query,
    calls,
    mean_time,
    rows,
    shared_blks_hit,
    shared_blks_read
FROM pg_stat_statements
WHERE queryid = <queryid>;
```

### 3.6. Сброс статистики

**Задание 3.5:** Выполните сброс статистики для тестирования:

```sql
-- Сброс всей статистики
SELECT pg_stat_statements_reset();

-- Проверка после сброса
SELECT count(*) FROM pg_stat_statements;

-- Выполните новый запрос и проверьте статистику
SELECT * FROM test_data WHERE value = 500;
SELECT queryid, calls, total_time, rows, query 
FROM pg_stat_statements 
WHERE query LIKE '%test_data%' 
ORDER BY total_time DESC;
```


## Часть 4. Комплексный мониторинг 

**Задание 4.1:** Создайте комплексное представление для мониторинга:

```sql
-- Создаём таблицу для хранения истории мониторинга
CREATE TABLE monitoring_history (
    check_time TIMESTAMP WITH TIME ZONE DEFAULT NOW() PRIMARY KEY,
    total_sessions INTEGER,
    active_sessions INTEGER,
    idle_transaction_sessions INTEGER,
    avg_connection_age_seconds FLOAT,
    max_connection_age_seconds FLOAT,
    total_queries_logged INTEGER,
    avg_query_time_ms FLOAT,
    total_blocks_read BIGINT,
    total_temp_blocks BIGINT
);

-- Функция для сбора метрик
CREATE OR REPLACE FUNCTION collect_monitoring_metrics()
RETURNS VOID AS $$
DECLARE
    v_now TIMESTAMP := NOW();
BEGIN
    INSERT INTO monitoring_history (
        check_time,
        total_sessions,
        active_sessions,
        idle_transaction_sessions,
        avg_connection_age_seconds,
        max_connection_age_seconds,
        total_queries_logged,
        avg_query_time_ms,
        total_blocks_read,
        total_temp_blocks
    )
    SELECT
        v_now,
        count(*) AS total_sessions,
        count(*) FILTER (WHERE state = 'active') AS active_sessions,
        count(*) FILTER (WHERE state = 'idle in transaction') AS idle_transaction_sessions,
        COALESCE(avg(EXTRACT(EPOCH FROM (NOW() - backend_start))), 0) AS avg_connection_age,
        COALESCE(max(EXTRACT(EPOCH FROM (NOW() - backend_start))), 0) AS max_connection_age,
        (SELECT count(*) FROM pg_stat_statements) AS total_queries_logged,
        COALESCE(avg(mean_time), 0) AS avg_query_time_ms,
        COALESCE(sum(shared_blks_read), 0) AS total_blocks_read,
        COALESCE(sum(temp_blks_read + temp_blks_written), 0) AS total_temp_blocks
    FROM pg_stat_activity
    CROSS JOIN pg_stat_statements;
END;
$$ LANGUAGE plpgsql;

-- Сбор метрик
SELECT collect_monitoring_metrics();

-- Просмотр истории
SELECT * FROM monitoring_history ORDER BY check_time DESC;
```

**Задание 4.2:** Создайте представление для быстрого просмотра состояния системы:

```sql
-- Представление для быстрого мониторинга
CREATE VIEW system_health AS
SELECT 
    (SELECT count(*) FROM pg_stat_activity) AS total_connections,
    (SELECT count(*) FROM pg_stat_activity WHERE state = 'active') AS active_queries,
    (SELECT count(*) FROM pg_stat_activity WHERE state = 'idle in transaction') AS idle_in_transaction,
    (SELECT count(*) FROM pg_locks WHERE NOT granted) AS pending_locks,
    (SELECT count(*) FROM pg_stat_statements) AS cached_queries,
    (SELECT round(avg(mean_time), 2) FROM pg_stat_statements) AS avg_query_time_ms,
    (SELECT round(max(max_time), 2) FROM pg_stat_statements) AS max_query_time_ms,
    (SELECT sum(calls) FROM pg_stat_statements) AS total_queries_executed;

-- Просмотр
SELECT * FROM system_health;
```


## Часть 5. Интеграция логирования и мониторинга

**Задание 5.1:** Настройте логирование queryid для связи логов со статистикой:

```sql
-- Добавляем queryid в формат логов
ALTER SYSTEM SET log_line_prefix = '%m [%p] %u@%d %r %a %Q ';
SELECT pg_reload_conf();

-- Теперь в логах будет отображаться queryid
-- Проверка (выполните запрос и проверьте лог)
SELECT pg_sleep(0.1);
SELECT * FROM test_data WHERE id = 1;
```

**Задание 5.2:** Создайте запрос для связи логов с pg_stat_statements:

```sql
-- Поиск запроса по queryid в статистике
SELECT 
    queryid,
    calls,
    total_time,
    mean_time,
    query
FROM pg_stat_statements
WHERE queryid = <queryid_from_log>;
```



##  Очистка окружения

```sql
-- Удаление созданных объектов
DROP TABLE IF EXISTS test_data CASCADE;
DROP TABLE IF EXISTS log_test_table CASCADE;
DROP TABLE IF EXISTS monitoring_history CASCADE;
DROP VIEW IF EXISTS session_monitor CASCADE;
DROP VIEW IF EXISTS system_health CASCADE;
DROP FUNCTION IF EXISTS slow_query() CASCADE;
DROP FUNCTION IF EXISTS collect_monitoring_metrics() CASCADE;

-- Сброс настроек логирования к значениям по умолчанию
ALTER SYSTEM RESET logging_collector;
ALTER SYSTEM RESET log_destination;
ALTER SYSTEM RESET log_directory;
ALTER SYSTEM RESET log_filename;
ALTER SYSTEM RESET log_file_mode;
ALTER SYSTEM RESET log_rotation_age;
ALTER SYSTEM RESET log_rotation_size;
ALTER SYSTEM RESET log_truncate_on_rotation;
ALTER SYSTEM RESET log_min_messages;
ALTER SYSTEM RESET log_min_error_statement;
ALTER SYSTEM RESET log_min_duration_statement;
ALTER SYSTEM RESET log_statement;
ALTER SYSTEM RESET log_line_prefix;
ALTER SYSTEM RESET log_connections;
ALTER SYSTEM RESET log_disconnections;
ALTER SYSTEM RESET log_checkpoints;
ALTER SYSTEM RESET log_lock_waits;

SELECT pg_reload_conf();

-- Переключение на другую базу и удаление
\c postgres
DROP DATABASE IF EXISTS lab_monitoring;
```
