# Лабораторная работа: Управление доступом (RBAC) и защита на уровне строк (RLS)

**Цель работы:** Изучить на практике ролевую модель управления доступом (Role-Based Access Control, RBAC) и механизм защиты на уровне строк (Row-Level Security, RLS) в PostgreSQL/Postgres Pro. Научиться создавать роли, управлять привилегиями, строить иерархии ролей и реализовывать политики доступа к отдельным строкам таблиц.


## Часть 0. Подготовка окружения (5 минут)

### 0.1. Создание базы данных для лабораторной

Выполните в терминале (под пользователем `postgres`):

```bash
sudo -u postgres psql
```

```sql
-- Создаём базу данных для лабораторной работы
CREATE DATABASE lab_rbac_rls;

-- Подключаемся к ней
\c lab_rbac_rls
```

---

## Часть 1. Роли и базовые привилегии (RBAC) 

### 1.1. Создание ролей с различными атрибутами

**Задание 1.1:** Создайте роли для различных уровней доступа в системе интернет-магазина:

```sql
-- 1. Групповая роль "читатели" (без права входа)
CREATE ROLE readers NOLOGIN;

-- 2. Групповая роль "писатели" (без права входа)
CREATE ROLE writers NOLOGIN;

-- 3. Групповая роль "администраторы" (без права входа)
CREATE ROLE admins NOLOGIN;

-- 4. Роль-пользователь "аналитик" (с правом входа)
CREATE ROLE analyst LOGIN PASSWORD 'analyst123';

-- 5. Роль-пользователь "менеджер" (с правом входа)
CREATE ROLE manager LOGIN PASSWORD 'manager123';

-- 6. Роль-пользователь "админ" (с правами CREATEDB и CREATEROLE)
CREATE ROLE admin_user LOGIN CREATEDB CREATEROLE PASSWORD 'admin123';
```

**Задание 1.2:** Проверьте созданные роли:

```sql
-- Просмотр всех ролей в кластере
\du

-- Детальная информация о ролях
SELECT rolname, rolsuper, rolcreatedb, rolcreaterole, rolcanlogin, rolreplication
FROM pg_roles
ORDER BY rolname;
```

### 1.2. Иерархия ролей и наследование

**Задание 1.3:** Постройте иерархию ролей:

```sql
-- writers наследуют права readers (писатели могут читать)
GRANT readers TO writers;

-- admins наследуют права writers (администраторы могут читать и писать)
GRANT writers TO admins;

-- Назначаем пользователей в группы
GRANT readers TO analyst;      -- аналитик только читает
GRANT writers TO manager;      -- менеджер читает и пишет
GRANT admins TO admin_user;    -- админ имеет все права группы admins
```

**Задание 1.4:** Проверьте членство в ролях:

```sql
-- Кто входит в какие роли
SELECT 
    r.rolname AS role_name,
    ARRAY(SELECT m.rolname 
          FROM pg_auth_members am 
          JOIN pg_roles m ON am.member = m.oid 
          WHERE am.roleid = r.oid) AS members
FROM pg_roles r
WHERE r.rolname IN ('readers', 'writers', 'admins', 'analyst', 'manager', 'admin_user')
ORDER BY r.rolname;
```

### 1.3. Создание объектов и назначение привилегий

**Задание 1.5:** Создайте таблицы для интернет-магазина:

```sql
-- Таблица товаров
CREATE TABLE products (
    id SERIAL PRIMARY KEY,
    name TEXT NOT NULL,
    price DECIMAL(10,2) NOT NULL,
    stock INTEGER DEFAULT 0,
    created_by TEXT
);

-- Таблица заказов
CREATE TABLE orders (
    id SERIAL PRIMARY KEY,
    product_id INTEGER REFERENCES products(id),
    customer_name TEXT NOT NULL,
    quantity INTEGER NOT NULL,
    order_date TIMESTAMP DEFAULT NOW(),
    status TEXT DEFAULT 'new'
);

-- Вставка тестовых данных
INSERT INTO products (name, price, stock, created_by) VALUES
    ('Ноутбук', 50000, 10, 'admin_user'),
    ('Мышь', 1000, 50, 'admin_user'),
    ('Клавиатура', 2500, 30, 'admin_user');

INSERT INTO orders (product_id, customer_name, quantity, status) VALUES
    (1, 'Иванов Иван', 1, 'new'),
    (2, 'Петров Пётр', 2, 'processing'),
    (3, 'Сидоров Сидор', 1, 'shipped');
```

**Задание 1.6:** Назначьте привилегии ролям:

```sql
-- readers: только чтение всех таблиц
GRANT SELECT ON products, orders TO readers;

-- writers: чтение + изменение
GRANT SELECT, INSERT, UPDATE, DELETE ON products, orders TO writers;

-- admins: все привилегии (включая изменение схемы)
GRANT ALL PRIVILEGES ON products, orders TO admins;
GRANT USAGE, SELECT ON SEQUENCE products_id_seq, orders_id_seq TO writers;

-- Права по умолчанию для будущих таблиц
ALTER DEFAULT PRIVILEGES IN SCHEMA public GRANT SELECT ON TABLES TO readers;
ALTER DEFAULT PRIVILEGES IN SCHEMA public GRANT SELECT, INSERT, UPDATE, DELETE ON TABLES TO writers;
```

**Задание 1.7:** Проверьте работу привилегий (подключитесь от имени разных пользователей):

```sql
-- Подключение как analyst (только чтение)
-- В отдельном терминале: psql -U analyst -d lab_rbac_rls
-- Или используйте SET ROLE в текущей сессии

SET ROLE analyst;
SELECT * FROM products;  -- должно работать
INSERT INTO products (name, price) VALUES ('Монитор', 15000);  -- должно выдать ошибку
RESET ROLE;

-- Подключение как manager (чтение и запись)
SET ROLE manager;
SELECT * FROM products;  -- работает
INSERT INTO products (name, price) VALUES ('Монитор', 15000);  -- работает
RESET ROLE;
```

### 1.4. Атрибут INHERIT и его влияние

**Задание 1.8:** Изучите влияние атрибута `INHERIT`:

```sql
-- Создаём роль без наследования
CREATE ROLE no_inherit_user LOGIN NOINHERIT PASSWORD 'noinherit123';

-- Назначаем в группу readers
GRANT readers TO no_inherit_user;

-- Проверяем: без SET ROLE права не наследуются
SET ROLE no_inherit_user;
SELECT * FROM products;  -- ОШИБКА: нет прав (если NOINHERIT)
RESET ROLE;

-- Явное переключение на роль readers
SET ROLE no_inherit_user;
SET ROLE readers;
SELECT * FROM products;  -- Работает
RESET ROLE;
RESET ROLE;
```

**Задание 1.9:** Сравните с ролью, у которой `INHERIT` включён (по умолчанию):

```sql
-- Создаём роль с наследованием (по умолчанию)
CREATE ROLE inherit_user LOGIN PASSWORD 'inherit123';
GRANT readers TO inherit_user;

-- Проверяем: права наследуются автоматически
SET ROLE inherit_user;
SELECT * FROM products;  -- Работает без SET ROLE
RESET ROLE;
```

---

## Часть 2. Row-Level Security (RLS)

### 2.1. Простой пример: фильтрация "мёртвых" записей

**Задание 2.1:** Создайте таблицу с "мягким" удалением:

```sql
-- Таблица клиентов с флагом удаления
CREATE TABLE customers (
    id SERIAL PRIMARY KEY,
    name TEXT NOT NULL,
    email TEXT,
    phone TEXT,
    deadfiled BOOLEAN DEFAULT FALSE,
    created_at TIMESTAMP DEFAULT NOW()
);

-- Вставка данных
INSERT INTO customers (name, email, phone, deadfiled) VALUES
    ('Иванов Иван', 'ivan@mail.ru', '+79991112233', FALSE),
    ('Петров Пётр', 'petr@mail.ru', '+79994445566', FALSE),
    ('Сидоров Сидор', 'sidor@mail.ru', '+79997778899', TRUE);  -- "удалён"

-- Создаём роль staff
CREATE ROLE staff LOGIN PASSWORD 'staff123';
GRANT SELECT, INSERT, UPDATE, DELETE ON customers TO staff;
GRANT USAGE ON SEQUENCE customers_id_seq TO staff;
```

**Задание 2.2:** Включите RLS и создайте политику:

```sql
-- Включаем RLS для таблицы
ALTER TABLE customers ENABLE ROW LEVEL SECURITY;

-- Создаём политику: staff не видит "мёртвые" записи
CREATE POLICY filter_deadfiled ON customers
    FOR SELECT TO staff
    USING (deadfiled IS NOT TRUE);
```

**Задание 2.3:** Проверьте работу политики:

```sql
-- Подключение как staff
SET ROLE staff;

-- Видны только строки с deadfiled = FALSE
SELECT * FROM customers;  -- не видно Сидорова

-- Попытка обновить видимую строку (должна работать)
UPDATE customers SET phone = '+79990001122' WHERE id = 1;

-- Попытка обновить невидимую строку (не должна работать)
UPDATE customers SET phone = '+79990001122' WHERE id = 3;  -- 0 rows updated

RESET ROLE;
```

**Задание 2.4:** Добавьте политику для всех операций (не только SELECT):

```sql
-- Удаляем старую политику
DROP POLICY filter_deadfiled ON customers;

-- Создаём политику для всех операций
CREATE POLICY filter_deadfiled_all ON customers
    FOR ALL TO staff
    USING (deadfiled IS NOT TRUE);
```

Проверьте, что теперь staff может выполнять `INSERT`, `UPDATE`, `DELETE` над видимыми строками.

### 2.2. Мультиарендная изоляция данных

Это наиболее частый сценарий использования RLS.

**Задание 2.5:** Создайте мультиарендную структуру:

```sql
-- Таблица арендаторов (tenants)
CREATE TABLE tenants (
    id SERIAL PRIMARY KEY,
    name TEXT NOT NULL,
    db_user TEXT UNIQUE NOT NULL   -- роль базы данных для арендатора
);

-- Таблица заказов арендаторов
CREATE TABLE tenant_orders (
    id SERIAL PRIMARY KEY,
    tenant_id INTEGER NOT NULL REFERENCES tenants(id),
    order_number TEXT NOT NULL,
    amount DECIMAL(10,2) NOT NULL,
    status TEXT DEFAULT 'new',
    created_at TIMESTAMP DEFAULT NOW()
);

-- Создаём роли для арендаторов
CREATE ROLE tenant1 LOGIN PASSWORD 'tenant1pass';
CREATE ROLE tenant2 LOGIN PASSWORD 'tenant2pass';

-- Добавляем арендаторов
INSERT INTO tenants (name, db_user) VALUES
    ('Компания "Альфа"', 'tenant1'),
    ('Компания "Бета"', 'tenant2');

-- Вставляем тестовые заказы
INSERT INTO tenant_orders (tenant_id, order_number, amount) VALUES
    (1, 'ALPHA-001', 15000.00),
    (1, 'ALPHA-002', 22000.00),
    (2, 'BETA-001', 8000.00),
    (2, 'BETA-002', 12000.00);

-- Даём базовые права
GRANT SELECT, INSERT, UPDATE, DELETE ON tenant_orders TO tenant1, tenant2;
GRANT USAGE ON SEQUENCE tenant_orders_id_seq TO tenant1, tenant2;
```

**Задание 2.6:** Создайте RLS-политику для изоляции арендаторов:

```sql
-- Включаем RLS
ALTER TABLE tenant_orders ENABLE ROW LEVEL SECURITY;

-- Политика: каждый арендатор видит только свои данные
CREATE POLICY tenant_isolation ON tenant_orders
    FOR ALL
    TO tenant1, tenant2
    USING (tenant_id = (SELECT id FROM tenants WHERE db_user = current_user))
    WITH CHECK (tenant_id = (SELECT id FROM tenants WHERE db_user = current_user));
```

**Разбор политики**:

| Часть | Значение |
|-------|----------|
| `FOR ALL` | Применяется ко всем командам |
| `TO tenant1, tenant2` | Применяется к ролям арендаторов |
| `USING (...)` | При `SELECT`, `UPDATE`, `DELETE` видимы только строки с `tenant_id` текущего арендатора |
| `WITH CHECK (...)` | При `INSERT` или `UPDATE` новая строка должна принадлежать текущему арендатору |

**Задание 2.7:** Проверьте изоляцию данных:

```sql
-- Подключение как tenant1
SET ROLE tenant1;
SELECT * FROM tenant_orders;  -- только заказы Альфы
INSERT INTO tenant_orders (order_number, amount) VALUES ('ALPHA-003', 5000.00);  -- tenant_id подставится автоматически?
-- Покажет ошибку, потому что tenant_id не указан
RESET ROLE;

-- Правильная вставка от имени арендатора
SET ROLE tenant1;
INSERT INTO tenant_orders (tenant_id, order_number, amount) 
VALUES ((SELECT id FROM tenants WHERE db_user = current_user), 'ALPHA-003', 5000.00);
SELECT * FROM tenant_orders;
RESET ROLE;
```

### 2.3. PERMISSIVE и RESTRICTIVE политики

**Задание 2.8:** Изучите комбинирование политик:

```sql
-- Таблица сотрудников
CREATE TABLE employees (
    id SERIAL PRIMARY KEY,
    name TEXT NOT NULL,
    department TEXT NOT NULL,
    salary INTEGER,
    is_active BOOLEAN DEFAULT TRUE
);

INSERT INTO employees (name, department, salary) VALUES
    ('Иванов', 'IT', 100000),
    ('Петров', 'IT', 80000),
    ('Сидоров', 'HR', 70000),
    ('Козлов', 'HR', 65000);

-- Роли
CREATE ROLE hr_user LOGIN PASSWORD 'hr123';
CREATE ROLE it_user LOGIN PASSWORD 'it123';
CREATE ROLE director LOGIN PASSWORD 'dir123';

GRANT SELECT ON employees TO hr_user, it_user, director;
```

**PERMISSIVE политики** (объединяются через OR):

```sql
ALTER TABLE employees ENABLE ROW LEVEL SECURITY;

-- PERMISSIVE: сотрудник видит только свой отдел
CREATE POLICY dept_policy ON employees
    FOR SELECT
    TO hr_user, it_user
    USING (department = CASE 
        WHEN current_user = 'hr_user' THEN 'HR'
        WHEN current_user = 'it_user' THEN 'IT'
    END);

-- PERMISSIVE: все видят активных сотрудников
CREATE POLICY active_policy ON employees
    FOR SELECT
    TO hr_user, it_user, director
    USING (is_active = TRUE);
```

Проверьте: пользователь из IT видит только активных сотрудников IT-отдела (объединение через OR).

**RESTRICTIVE политика** (объединяются через AND):

```sql
-- RESTRICTIVE: директор видит всех, но с ограничением по зарплате
CREATE POLICY salary_restriction ON employees
    FOR SELECT
    TO director
    AS RESTRICTIVE
    USING (salary < 150000);

-- Проверка
SET ROLE director;
SELECT * FROM employees;  -- не видит сотрудников с зарплатой >= 150000
RESET ROLE;
```

### 2.4. BYPASSRLS — обход RLS

**Задание 2.9:** Создайте роль, которая обходит RLS:

```sql
-- Только суперпользователь может создать BYPASSRLS
CREATE ROLE admin_bypass BYPASSRLS LOGIN PASSWORD 'bypass123';

-- Проверка: admin_bypass видит все строки, включая "мёртвые"
GRANT SELECT ON customers TO admin_bypass;

SET ROLE admin_bypass;
SELECT * FROM customers;  -- видит все строки, включая deadfiled = TRUE
RESET ROLE;
```

---

## Часть 3. Просмотр и диагностика RLS 

**Задание 3.1:** Просмотр политик:

```sql
-- Все политики в базе данных
\dp

-- Политики для конкретной таблицы
SELECT * FROM pg_policies WHERE tablename = 'customers';
SELECT * FROM pg_policies WHERE tablename = 'tenant_orders';
```

**Задание 3.2:** Проверка состояния RLS для таблиц:

```sql
SELECT 
    relname,
    relrowsecurity AS rls_enabled,
    relforcerowsecurity AS rls_forced
FROM pg_class
WHERE relname IN ('customers', 'tenant_orders', 'employees', 'products');
```

**Задание 3.3:** Временное отключение RLS для сессии (только для BYPASSRLS или суперпользователя):

```sql
-- Только для административных целей
SET row_security = off;
SELECT * FROM customers;  -- видит все строки
SET row_security = on;
```


## Очистка окружения

```sql
-- Переключение на другую базу
\c postgres

-- Удаление базы данных (удалит все объекты)
DROP DATABASE IF EXISTS lab_rbac_rls;

-- Если нужно удалить роли (глобальные объекты)
DROP ROLE IF EXISTS analyst, manager, admin_user, staff, tenant1, tenant2;
DROP ROLE IF EXISTS hr_user, it_user, director, admin_bypass;
DROP ROLE IF EXISTS project_manager, developer, viewer;
DROP ROLE IF EXISTS readers, writers, admins;
DROP ROLE IF EXISTS no_inherit_user, inherit_user;
```

---

**Успешной работы!**
