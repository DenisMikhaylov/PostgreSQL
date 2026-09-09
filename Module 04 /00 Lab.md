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
