# Лабораторная работа: Резервное копирование и восстановление в PostgreSQL


**Цель работы:** Освоить логическое и физическое резервное копирование PostgreSQL, научиться настраивать WAL-архивирование для PITR, сравнивать форматы дампов, автоматизировать бэкапы через cron, восстанавливать отдельные таблицы и схемы, а также использовать параллельное выполнение для ускорения операций.
---

## Часть 0. Подготовка окружения 

### 0.1. Проверка установки утилит

**Что делаем:** Убеждаемся, что все необходимые утилиты доступны.

**Почему:** Без них лабораторная не выполнится.

```bash
# Проверка версий утилит
pg_dump --version
pg_dumpall --version
pg_restore --version
pg_basebackup --version
psql --version

# Проверка пользователя postgres
sudo -u postgres whoami
```

### 0.2. Создание тестовой базы данных

**Что делаем:** Создаём базу `lab_backup` с несколькими схемами и таблицами.

**Почему:** Разные схемы и таблицы нужны, чтобы продемонстрировать выборочное восстановление.

```bash
sudo -u postgres psql
```

```sql
CREATE DATABASE lab_backup;

\c lab_backup

-- Схема sales
CREATE SCHEMA sales;

CREATE TABLE sales.customers (
    customer_id SERIAL PRIMARY KEY,
    name        TEXT NOT NULL,
    email       TEXT,
    city        TEXT,
    created_at  TIMESTAMP DEFAULT NOW()
);

CREATE TABLE sales.orders (
    order_id    SERIAL PRIMARY KEY,
    customer_id INTEGER REFERENCES sales.customers(customer_id),
    amount      NUMERIC(10,2),
    order_date  DATE DEFAULT CURRENT_DATE
);

-- Схема hr
CREATE SCHEMA hr;

CREATE TABLE hr.employees (
    emp_id     SERIAL PRIMARY KEY,
    emp_name   TEXT NOT NULL,
    department TEXT,
    salary     NUMERIC(10,2)
);

-- Схема public
CREATE TABLE public.products (
    product_id SERIAL PRIMARY KEY,
    name       TEXT NOT NULL,
    price      NUMERIC(10,2)
);
```

### 0.3. Заполнение данными

```sql
-- sales.customers
INSERT INTO sales.customers (name, email, city)
SELECT 
    'Customer ' || g,
    'customer' || g || '@example.com',
    (ARRAY['Москва','Санкт-Петербург','Казань'])[1 + (g % 3)]
FROM generate_series(1, 10000) AS g;

-- sales.orders
INSERT INTO sales.orders (customer_id, amount, order_date)
SELECT 
    1 + (random() * 9999)::int,
    round((random() * 10000 + 100)::numeric, 2),
    DATE '2024-01-01' + (random() * 365)::int
FROM generate_series(1, 50000);

-- hr.employees
INSERT INTO hr.employees (emp_name, department, salary)
SELECT 
    'Employee ' || g,
    (ARRAY['IT','HR','Sales','Finance'])[1 + (g % 4)],
    round((random() * 200000 + 50000)::numeric, 2)
FROM generate_series(1, 1000) AS g;

-- public.products
INSERT INTO public.products (name, price)
SELECT 
    'Product ' || g,
    round((random() * 5000 + 100)::numeric, 2)
FROM generate_series(1, 500) AS g;
```

### 0.4. Создание каталогов для бэкапов

**Что делаем:** Создаём каталоги для разных типов бэкапов.

**Почему:** Чёткая структура упрощает управление и автоматизацию.

```bash
sudo mkdir -p /var/backups/postgresql/logical
sudo mkdir -p /var/backups/postgresql/physical
sudo mkdir -p /var/backups/postgresql/wal_archive
sudo chown -R postgres:postgres /var/backups/postgresql
sudo chmod 700 /var/backups/postgresql

ls -la /var/backups/postgresql/
```


## 📋 Часть 1. Логическое резервное копирование (35 минут)

### 1.1. pg_dump в формате plain (SQL)

**Что делаем:** Создаём дамп в plain SQL.

**Почему:** Это самый простой формат — текстовый файл с SQL-командами, который можно читать и редактировать.

```bash
sudo -u postgres pg_dump -d lab_backup -Fp -f /var/backups/postgresql/logical/lab_backup_plain.sql

# Проверяем
ls -lh /var/backups/postgresql/logical/lab_backup_plain.sql
head -50 /var/backups/postgresql/logical/lab_backup_plain.sql
```

**Что смотреть в файле:**
- `CREATE SCHEMA`, `CREATE TABLE` — DDL-команды
- `COPY` или `INSERT` — данные
- Комментарии и настройки

**Размер файла:** обратите внимание — plain SQL самый большой по размеру (без сжатия).

### 1.2. pg_dump в формате custom

**Что делаем:** Создаём дамп в custom-формате со сжатием.

**Почему:** Custom — рекомендуемый формат для production: сжатие + выборочное восстановление.

```bash
sudo -u postgres pg_dump -d lab_backup -Fc -f /var/backups/postgresql/logical/lab_backup_custom.dump

ls -lh /var/backups/postgresql/logical/lab_backup_custom.dump
```

**Сравните размеры:**
```bash
ls -lh /var/backups/postgresql/logical/
```

**Ожидаемый результат:** custom-файл в 3–5 раз меньше plain.

**Просмотр оглавления архива:**
```bash
sudo -u postgres pg_restore -l /var/backups/postgresql/logical/lab_backup_custom.dump | head -30
```

### 1.3. pg_dump в формате directory (параллельный)

**Что делаем:** Создаём дамп в directory-формате с параллелизмом.

**Почему:** Directory — **единственный формат, поддерживающий параллельный дамп** (`-j`).

```bash
# Обычный directory-дамп
sudo -u postgres pg_dump -d lab_backup -Fd -f /var/backups/postgresql/logical/lab_backup_dir

# Параллельный дамп (4 процесса)
sudo -u postgres pg_dump -d lab_backup -Fd -j 4 -f /var/backups/postgresql/logical/lab_backup_dir_parallel

# Смотрим структуру
ls -la /var/backups/postgresql/logical/lab_backup_dir_parallel/
```

**Что смотреть:** в directory-формате создаётся **несколько файлов** (по одному на каждый объект). При параллельном дампе файлы создаются одновременно.

**Замер времени:**
```bash
# Без параллелизма
time sudo -u postgres pg_dump -d lab_backup -Fd -f /tmp/test_single

# С параллелизмом
time sudo -u postgres pg_dump -d lab_backup -Fd -j 4 -f /tmp/test_parallel

# Очистка
rm -rf /tmp/test_single /tmp/test_parallel
```

### 1.4. pg_dump в формате tar

**Что делаем:** Создаём дамп в tar-формате.

**Почему:** Tar-формат — это упакованная директория, удобная для передачи по сети.

```bash
sudo -u postgres pg_dump -d lab_backup -Ft -f /var/backups/postgresql/logical/lab_backup.tar

ls -lh /var/backups/postgresql/logical/lab_backup.tar
```

### 1.5. pg_dumpall — весь кластер

**Что делаем:** Создаём дамп всего кластера.

**Почему:** `pg_dumpall` включает **глобальные объекты** (роли, табличные пространства), которые не попадают в `pg_dump`.

```bash
# Полный дамп кластера
sudo -u postgres pg_dumpall -f /var/backups/postgresql/logical/cluster_full.sql

# Только глобальные объекты (роли, табличные пространства)
sudo -u postgres pg_dumpall --globals-only -f /var/backups/postgresql/logical/globals.sql

ls -lh /var/backups/postgresql/logical/cluster_full.sql /var/backups/postgresql/logical/globals.sql
head -30 /var/backups/postgresql/logical/globals.sql
```



## Часть 2. Восстановление из логических бэкапов

### 2.1. Восстановление plain SQL через psql

**Что делаем:** Восстанавливаем базу из plain SQL.

**Почему:** Plain SQL восстанавливается через `psql`, а не через `pg_restore`.

```sql
-- Создаём новую БД для восстановления
CREATE DATABASE lab_restore_plain;
```

```bash
# Восстановление
sudo -u postgres psql -d lab_restore_plain -f /var/backups/postgresql/logical/lab_backup_plain.sql 2>&1 | tail -5
```

**Проверка:**
```bash
sudo -u postgres psql -d lab_restore_plain -c "\dt sales.*"
sudo -u postgres psql -d lab_restore_plain -c "SELECT count(*) FROM sales.customers;"
```

### 2.2. Восстановление custom-дампа

**Что делаем:** Восстанавливаем базу из custom-дампа.

**Почему:** Custom-формат — рекомендуемый, поддерживает выборочное восстановление.

```sql
CREATE DATABASE lab_restore_custom;
```

```bash
# Восстановление с созданием объектов
sudo -u postgres pg_restore -d lab_restore_custom /var/backups/postgresql/logical/lab_backup_custom.dump

# Проверка
sudo -u postgres psql -d lab_restore_custom -c "\dt sales.*"
sudo -u postgres psql -d lab_restore_custom -c "SELECT count(*) FROM sales.orders;"
```

### 2.3. Выборочное восстановление отдельной таблицы

**Что делаем:** Восстанавливаем только одну таблицу.

**Почему:** Это ключевое преимущество логического копирования — можно восстановить конкретный объект, не затрагивая остальные.

```sql
-- Создаём пустую БД
CREATE DATABASE lab_restore_table;
```

```bash
# Восстанавливаем только таблицу hr.employees
sudo -u postgres pg_restore -d lab_restore_table -t employees /var/backups/postgresql/logical/lab_backup_custom.dump

# Проверка
sudo -u postgres psql -d lab_restore_table -c "\dt"
sudo -u postgres psql -d lab_restore_table -c "SELECT count(*) FROM employees;"
```

**Обратите внимание:** восстановилась **только** таблица `employees`, но не схема `hr` целиком.

### 2.4. Выборочное восстановление схемы

**Что делаем:** Восстанавливаем только схему `sales`.

```sql
CREATE DATABASE lab_restore_schema;
```

```bash
# Восстанавливаем только схему sales
sudo -u postgres pg_restore -d lab_restore_schema -n sales /var/backups/postgresql/logical/lab_backup_custom.dump

# Проверка
sudo -u postgres psql -d lab_restore_schema -c "\dn"
sudo -u postgres psql -d lab_restore_schema -c "\dt sales.*"
```

### 2.5. Восстановление только данных (без схемы)

**Что делаем:** Восстанавливаем только данные в уже существующую таблицу.

**Почему:** Полезно при восстановлении после случайного `DELETE` или `UPDATE`.

```sql
-- Создаём БД и пустую таблицу
CREATE DATABASE lab_restore_data;
\c lab_restore_data

CREATE SCHEMA sales;
CREATE TABLE sales.customers (
    customer_id SERIAL PRIMARY KEY,
    name        TEXT NOT NULL,
    email       TEXT,
    city        TEXT,
    created_at  TIMESTAMP DEFAULT NOW()
);
```

```bash
# Восстанавливаем только данные
sudo -u postgres pg_restore -d lab_restore_data -a -t customers /var/backups/postgresql/logical/lab_backup_custom.dump

sudo -u postgres psql -d lab_restore_data -c "SELECT count(*) FROM sales.customers;"
```

## Часть 3. Физическое резервное копирование

### 3.1. Настройка параметров для физического бэкапа

**Что делаем:** Проверяем текущие настройки WAL.

**Почему:** Для физического бэкапа и PITR нужны определённые настройки.

```bash
sudo -u postgres psql -c "SHOW wal_level;"
sudo -u postgres psql -c "SHOW archive_mode;"
sudo -u postgres psql -c "SHOW max_wal_senders;"
```

**Ожидаемый вывод:**
```
wal_level      | replica      (по умолчанию в PostgreSQL 10+)
archive_mode   | off
max_wal_senders| 10
```

### 3.2. Создание слота репликации

**Что делаем:** Создаём слот репликации для `pg_basebackup`.

**Почему:** Слот гарантирует, что WAL-файлы не будут удалены до их передачи.

```bash
sudo -u postgres psql -c "SELECT * FROM pg_create_physical_replication_slot('backup_slot');"
```

### 3.3. Физический бэкап с pg_basebackup

**Что делаем:** Создаём физический бэкап кластера.

**Почему:** Физический бэкап — основа для PITR и быстрого восстановления всего кластера.

```bash
# Бэкап в формате tar со сжатием
sudo -u postgres pg_basebackup \
    -D /var/backups/postgresql/physical/base_$(date +%Y-%m-%d) \
    -Ft -z -P -X stream -c fast

# Смотрим результат
ls -lh /var/backups/postgresql/physical/base_$(date +%Y-%m-%d)/
```

**Что смотреть в выводе:**
- `transaction log start point` — LSN начала бэкапа
- `transaction log end point` — LSN конца бэкапа
- `backup completed` — успешное завершение

### 3.4. Проверка физического бэкапа

**Что делаем:** Проверяем целостность архива.

```bash
# Список файлов в архиве
tar -tzf /var/backups/postgresql/physical/base_$(date +%Y-%m-%d)/base.tar.gz | head -20

# Проверка наличия backup_label (ключевой файл для восстановления)
tar -tzf /var/backups/postgresql/physical/base_$(date +%Y-%m-%d)/base.tar.gz | grep backup_label
```

### 3.5. Очистка слота

```bash
sudo -u postgres psql -c "SELECT pg_drop_replication_slot('backup_slot');"
```


## Часть 4. PITR — точечное восстановление 

### 4.1. Настройка WAL-архивирования

**Что делаем:** Включаем `archive_mode` и настраиваем `archive_command`.

**Почему:** Без WAL-архива невозможно восстановление на произвольный момент времени.

```bash
# Смотрим текущий конфиг
sudo -u postgres psql -c "SHOW config_file;"
```

**Редактируем `postgresql.conf`:**
```bash
sudo -u postgres nano /etc/postgresql/*/main/postgresql.conf
```

**Добавляем/изменяем:**
```bash
wal_level = replica
archive_mode = on
archive_command = 'test ! -f /var/backups/postgresql/wal_archive/%f && cp %p /var/backups/postgresql/wal_archive/%f'
archive_timeout = 60
```

**Важно:** `archive_mode` требует **перезапуска** сервера.

```bash
sudo systemctl restart postgresql

# Проверка
sudo -u postgres psql -c "SHOW archive_mode;"
sudo -u postgres psql -c "SHOW archive_command;"
```

### 4.2. Создание базового бэкапа для PITR

**Что делаем:** Создаём базовый бэкап.

**Почему:** PITR = базовый бэкап + WAL-архив.

```bash
sudo -u postgres pg_basebackup \
    -D /var/backups/postgresql/physical/pitr_base \
    -Ft -z -P -X stream -c fast

ls -lh /var/backups/postgresql/physical/pitr_base/
```

### 4.3. Генерация изменений

**Что делаем:** Создаём данные после бэкапа — это будет «целевая точка» восстановления.

```bash
sudo -u postgres psql -d lab_backup
```

```sql
-- Запоминаем текущее время
SELECT NOW() AS point_in_time;

-- Создаём новую таблицу и вставляем данные
CREATE TABLE pitr_test (
    id SERIAL PRIMARY KEY,
    value TEXT,
    created_at TIMESTAMP DEFAULT NOW()
);

INSERT INTO pitr_test (value) VALUES ('before_crash_1'), ('before_crash_2');

-- Ещё раз фиксируем время — это целевая точка
SELECT NOW() AS target_time;
```

**Запишите целевое время!** Оно понадобится для восстановления.

### 4.4. Принудительная архивация WAL

```bash
# Переключаем WAL-сегмент, чтобы он попал в архив
sudo -u postgres psql -c "SELECT pg_switch_wal();"

# Проверяем архив
ls -lh /var/backups/postgresql/wal_archive/ | tail -5

# Проверяем статус архивации
sudo -u postgres psql -c "SELECT * FROM pg_stat_archiver;"
```

### 4.5. Имитация сбоя

**Что делаем:** Удаляем данные, чтобы продемонстрировать PITR.

```sql
DROP TABLE pitr_test;
SELECT 'Данные потеряны' AS status;
```

### 4.6. Выполнение PITR

**Что делаем:** Восстанавливаем кластер на момент до удаления таблицы.

**Останавливаем PostgreSQL:**
```bash
sudo systemctl stop postgresql
```

**Сохраняем текущий каталог данных (на всякий случай):**
```bash
sudo mv /var/lib/postgresql/*/main /var/lib/postgresql/*/main_old
```

**Распаковываем базовый бэкап:**
```bash
sudo mkdir -p /var/lib/postgresql/*/main
cd /var/lib/postgresql/*/main
sudo tar -xzf /var/backups/postgresql/physical/pitr_base/base.tar.gz
sudo tar -xzf /var/backups/postgresql/physical/pitr_base/pg_wal.tar.gz -C pg_wal/
```

**Настраиваем восстановление:**
```bash
sudo touch /var/lib/postgresql/*/main/recovery.signal

# Добавляем в postgresql.conf
sudo -u postgres tee -a /var/lib/postgresql/*/main/postgresql.conf <<EOF
restore_command = 'cp /var/backups/postgresql/wal_archive/%f %p'
recovery_target_time = '2026-09-14 15:30:00'   # ← ваше целевое время
recovery_target_action = 'promote'
EOF
```

**Запускаем PostgreSQL:**
```bash
sudo systemctl start postgresql
sudo systemctl status postgresql
```

**Проверяем восстановление:**
```bash
sudo -u postgres psql -d lab_backup -c "SELECT * FROM pitr_test;"
```

**Ожидаемый результат:** таблица `pitr_test` существует, данные на месте.

### 4.7. Просмотр журнала восстановления

```bash
sudo tail -30 /var/log/postgresql/postgresql-*.log | grep -i "recovery\|restore"
```


## Часть 5. Автоматизация бэкапов с cron 

### 5.1. Создание скрипта логического бэкапа

**Что делаем:** Создаём скрипт для ежедневного бэкапа.

**Почему:** Автоматизация — обязательное требование production.

```bash
sudo tee /usr/local/bin/pg_logical_backup.sh <<'EOF'
#!/bin/bash
# Логический бэкап базы lab_backup

BACKUP_DIR="/var/backups/postgresql/logical"
DATE=$(date +%Y-%m-%d_%H-%M-%S)
DB_NAME="lab_backup"
RETENTION_DAYS=7

mkdir -p "$BACKUP_DIR"

# Custom-формат
pg_dump -U postgres -Fc -f "$BACKUP_DIR/${DB_NAME}_${DATE}.dump" "$DB_NAME"

if [ $? -eq 0 ]; then
    echo "$(date) - Backup SUCCESS: ${DB_NAME}_${DATE}.dump" >> "$BACKUP_DIR/backup.log"
else
    echo "$(date) - Backup FAILED: ${DB_NAME}" >> "$BACKUP_DIR/backup.log"
    exit 1
fi

# Удаление старых бэкапов
find "$BACKUP_DIR" -name "${DB_NAME}_*.dump" -mtime +$RETENTION_DAYS -delete
EOF

sudo chmod +x /usr/local/bin/pg_logical_backup.sh
sudo chown postgres:postgres /usr/local/bin/pg_logical_backup.sh
```

### 5.2. Тестирование скрипта

```bash
sudo -u postgres /usr/local/bin/pg_logical_backup.sh

ls -lh /var/backups/postgresql/logical/
cat /var/backups/postgresql/logical/backup.log
```

### 5.3. Создание скрипта физического бэкапа

```bash
sudo tee /usr/local/bin/pg_physical_backup.sh <<'EOF'
#!/bin/bash
# Физический бэкап кластера

BACKUP_DIR="/var/backups/postgresql/physical"
DATE=$(date +%Y-%m-%d)
RETENTION_DAYS=14

mkdir -p "$BACKUP_DIR"

pg_basebackup -U postgres -D "$BACKUP_DIR/base_$DATE" -Ft -z -P -X stream -c fast

if [ $? -eq 0 ]; then
    echo "$(date) - Base backup SUCCESS" >> "$BACKUP_DIR/backup.log"
else
    echo "$(date) - Base backup FAILED" >> "$BACKUP_DIR/backup.log"
    exit 1
fi

# Удаление старых бэкапов
find "$BACKUP_DIR" -maxdepth 1 -type d -name "base_*" -mtime +$RETENTION_DAYS -exec rm -rf {} \;
EOF

sudo chmod +x /usr/local/bin/pg_physical_backup.sh
sudo chown postgres:postgres /usr/local/bin/pg_physical_backup.sh
```

### 5.4. Настройка cron

**Что делаем:** Добавляем задания в cron.

**Почему:** Cron — стандартный инструмент автоматизации в Unix.

```bash
# Открываем crontab для postgres
sudo -u postgres crontab -e
```

**Добавляем строки:**
```cron
# Логический бэкап — ежедневно в 02:00
0 2 * * * /usr/local/bin/pg_logical_backup.sh >> /var/log/pg_logical_backup.log 2>&1

# Физический бэкап — ежедневно в 03:00
0 3 * * * /usr/local/bin/pg_physical_backup.sh >> /var/log/pg_physical_backup.log 2>&1
```

**Проверка:**
```bash
sudo -u postgres crontab -l
```

### 5.5. Проверка логов

```bash
sudo tail -20 /var/log/pg_logical_backup.log
sudo tail -20 /var/log/pg_physical_backup.log
```



## Очистка окружения

```sql
-- Удаление тестовых БД
DROP DATABASE IF EXISTS lab_restore_plain;
DROP DATABASE IF EXISTS lab_restore_custom;
DROP DATABASE IF EXISTS lab_restore_table;
DROP DATABASE IF EXISTS lab_restore_schema;
DROP DATABASE IF EXISTS lab_restore_data;
DROP DATABASE IF EXISTS lab_restore_parallel;
DROP DATABASE IF EXISTS lab_restore_full_schema;
DROP DATABASE IF EXISTS lab_backup;
```

```bash
# Удаление бэкапов
sudo rm -rf /var/backups/postgresql/

# Удаление скриптов
sudo rm -f /usr/local/bin/pg_logical_backup.sh
sudo rm -f /usr/local/bin/pg_physical_backup.sh

# Удаление заданий cron
sudo -u postgres crontab -r

# Отключение archive_mode (если нужно)
# Отредактируйте postgresql.conf: archive_mode = off
# sudo systemctl restart postgresql
```

**Успешной работы!**
