# Лабораторная работа: CTE (Common Table Expressions) в PostgreSQL



**Цель работы:** Изучить на практике обобщённые табличные выражения (Common Table Expressions, CTE) в PostgreSQL/Postgres Pro: синтаксис `WITH`, рекурсивные CTE


## Часть 0. Подготовка окружения (10 минут)

### 0.1. Создание базы данных

```sql
CREATE DATABASE lab_cte;

\c lab_cte
```

### 0.2. Создание тестовых таблиц

```sql
-- Сотрудники (с иерархией — manager_id ссылается на emp_id)
CREATE TABLE employees (
    emp_id     INTEGER PRIMARY KEY,
    emp_name   TEXT NOT NULL,
    manager_id INTEGER REFERENCES employees(emp_id),
    dept_id    INTEGER,
    position   TEXT,
    salary     NUMERIC(10,2),
    hired_at   DATE
);

-- Отделы (с иерархией — parent_dept_id ссылается на dept_id)
CREATE TABLE departments (
    dept_id        INTEGER PRIMARY KEY,
    dept_name      TEXT NOT NULL,
    parent_dept_id INTEGER REFERENCES departments(dept_id),
    city           TEXT
);

-- Продажи
CREATE TABLE sales (
    sale_id    SERIAL PRIMARY KEY,
    emp_id     INTEGER REFERENCES employees(emp_id),
    sale_date  DATE NOT NULL,
    amount     NUMERIC(10,2) NOT NULL,
    region     TEXT
);

-- Журнал изменений (для CTE с DML)
CREATE TABLE audit_log (
    log_id     SERIAL PRIMARY KEY,
    action     TEXT NOT NULL,
    emp_id     INTEGER,
    old_salary NUMERIC(10,2),
    new_salary NUMERIC(10,2),
    changed_at TIMESTAMP DEFAULT NOW()
);
```

### 0.3. Заполнение тестовыми данными

```sql
-- Отделы (иерархия: 1 → 2,3; 2 → 4,5)
INSERT INTO departments VALUES
    (1, 'Головной офис',   NULL, 'Москва'),
    (2, 'ИТ-департамент',  1,    'Москва'),
    (3, 'Маркетинг',       1,    'Санкт-Петербург'),
    (4, 'Разработка',      2,    'Москва'),
    (5, 'Поддержка',       2,    'Казань');

-- Сотрудники (иерархия менеджеров)
INSERT INTO employees VALUES
    (101, 'Иванов',   NULL, 1, 'CEO',              300000, '2015-01-10'),
    (102, 'Петров',   101,  2, 'CIO',              250000, '2016-03-15'),
    (103, 'Сидоров',  101,  3, 'CMO',              240000, '2017-06-01'),
    (104, 'Козлов',   102,  4, 'Dev Lead',         200000, '2018-09-10'),
    (105, 'Морозов',  102,  5, 'Support Lead',     180000, '2019-02-20'),
    (106, 'Новиков',  104,  4, 'Senior Developer', 170000, '2020-05-15'),
    (107, 'Орлов',    104,  4, 'Developer',        140000, '2021-08-01'),
    (108, 'Волков',   104,  4, 'Junior Developer', 100000, '2022-11-10'),
    (109, 'Лебедев',  105,  5, 'Support Engineer', 120000, '2021-04-05'),
    (110, 'Соколов',  105,  5, 'Support Engineer', 115000, '2022-07-20'),
    (111, 'Павлов',   103,  3, 'Marketer',          95000, '2022-01-15');

-- Продажи
INSERT INTO sales (emp_id, sale_date, amount, region)
SELECT 
    (ARRAY[101,102,103,104,105,106,107,108,109,110,111])[floor(random()*11+1)],
    DATE '2024-01-01' + (floor(random()*180) || ' days')::interval,
    round((random() * 50000 + 5000)::numeric, 2),
    (ARRAY['Москва','СПб','Казань','Новосибирск'])[floor(random()*4+1)]
FROM generate_series(1, 300);
```

### 0.4. Проверка данных

```sql
SELECT * FROM departments ORDER BY dept_id;
SELECT * FROM employees ORDER BY emp_id;
SELECT count(*), sum(amount) FROM sales;
```


## Часть 1. Базовый синтаксис CTE 

### 1.1. Что такое CTE?

**CTE (Common Table Expression)** — это именованный временный результат запроса, который существует в рамках **одного** SQL-выражения. CTE объявляется с помощью ключевого слова `WITH` и может быть использован в основном запросе как обычная таблица.

**Синтаксис:**

```sql
WITH имя_cte AS (
    SELECT ...
)
SELECT ... FROM имя_cte;
```

### 1.2. Простой CTE

**Задание 1.1:** Найдите отделы со средней зарплатой выше 150000.

```sql
WITH dept_avg AS (
    SELECT 
        dept_id,
        AVG(salary) AS avg_salary,
        COUNT(*)    AS emp_count
    FROM employees
    WHERE dept_id IS NOT NULL
    GROUP BY dept_id
)
SELECT 
    d.dept_name,
    da.avg_salary,
    da.emp_count
FROM dept_avg da
JOIN departments d ON da.dept_id = d.dept_id
WHERE da.avg_salary > 150000
ORDER BY da.avg_salary DESC;
```

**Задание 1.2:** Перепишите тот же запрос через подзапрос и сравните читаемость.

```sql
SELECT 
    d.dept_name,
    sub.avg_salary,
    sub.emp_count
FROM (
    SELECT 
        dept_id,
        AVG(salary) AS avg_salary,
        COUNT(*)    AS emp_count
    FROM employees
    WHERE dept_id IS NOT NULL
    GROUP BY dept_id
) sub
JOIN departments d ON sub.dept_id = d.dept_id
WHERE sub.avg_salary > 150000
ORDER BY sub.avg_salary DESC;
```


### 1.3. Несколько CTE в одном запросе

**Задание 1.3:** Используйте несколько CTE одновременно.

```sql
WITH 
dept_stats AS (
    SELECT 
        dept_id,
        AVG(salary) AS avg_salary,
        COUNT(*)    AS emp_count
    FROM employees
    WHERE dept_id IS NOT NULL
    GROUP BY dept_id
),
company_stats AS (
    SELECT 
        AVG(salary) AS company_avg_salary
    FROM employees
)
SELECT 
    d.dept_name,
    ds.avg_salary,
    cs.company_avg_salary,
    ROUND(ds.avg_salary - cs.company_avg_salary, 2) AS diff_from_company
FROM dept_stats ds
CROSS JOIN company_stats cs
JOIN departments d ON ds.dept_id = d.dept_id
ORDER BY diff_from_company DESC;
```


**Задание 1.4:** CTE, ссылающийся на предыдущий CTE.

```sql
WITH 
dept_stats AS (
    SELECT 
        dept_id,
        AVG(salary) AS avg_salary
    FROM employees
    GROUP BY dept_id
),
top_depts AS (
    SELECT * FROM dept_stats
    WHERE avg_salary > 150000
)
SELECT 
    d.dept_name, td.avg_salary
FROM top_depts td
JOIN departments d ON td.dept_id = d.dept_id;
```

### 1.4. CTE и алиасы столбцов

**Задание 1.5:** Задайте имена столбцов CTE явно.

```sql
WITH dept_summary (dept, avg_sal, total_emp) AS (
    SELECT 
        dept_id,
        AVG(salary),
        COUNT(*)
    FROM employees
    GROUP BY dept_id
)
SELECT d.dept_name, ds.avg_sal, ds.total_emp
FROM dept_summary ds
JOIN departments d ON ds.dept = d.dept_id;
```

### 1.5. CTE в подзапросах разных типов

**Задание 1.6:** CTE можно использовать в `FROM`, `JOIN`, `WHERE IN`, `EXISTS` и даже в `INSERT`/`UPDATE`/`DELETE`.

```sql
-- Использование в WHERE IN
WITH high_earners AS (
    SELECT emp_id FROM employees WHERE salary > 150000
)
SELECT * FROM sales
WHERE emp_id IN (SELECT emp_id FROM high_earners);

-- Использование в EXISTS
WITH dept_with_sales AS (
    SELECT DISTINCT e.dept_id
    FROM employees e
    JOIN sales s ON e.emp_id = s.emp_id
)
SELECT d.dept_name
FROM departments d
WHERE EXISTS (
    SELECT 1 FROM dept_with_sales dws WHERE dws.dept_id = d.dept_id
);
```





## Часть 2. Рекурсивные CTE

### 2.1. Что такое рекурсивный CTE?

**Рекурсивный CTE** — это CTE, который ссылается на себя. Он используется для обхода иерархических структур (деревьев, графов), генерации последовательностей и т.п.

**Синтаксис:**

```sql
WITH RECURSIVE имя AS (
    -- Базовая часть (anchor)
    SELECT ...
    
    UNION ALL
    
    -- Рекурсивная часть (ссылается на имя)
    SELECT ...
    FROM имя
    WHERE ...
)
SELECT ... FROM имя;
```


### 2.2. Обход иерархии сотрудников

**Задание 2.1:** Выведите всех сотрудников с указанием их уровня в иерархии.

```sql
WITH RECURSIVE emp_hierarchy AS (
    -- Базовая часть: CEO (manager_id IS NULL)
    SELECT 
        emp_id,
        emp_name,
        manager_id,
        1 AS level,
        emp_name::text AS path
    FROM employees
    WHERE manager_id IS NULL
    
    UNION ALL
    
    -- Рекурсивная часть: подчинённые
    SELECT 
        e.emp_id,
        e.emp_name,
        e.manager_id,
        h.level + 1,
        h.path || ' → ' || e.emp_name
    FROM employees e
    JOIN emp_hierarchy h ON e.manager_id = h.emp_id
)
SELECT 
    emp_id, 
    repeat('  ', level - 1) || emp_name AS org_chart,
    level,
    path
FROM emp_hierarchy
ORDER BY path;
```


### 2.3. Обход иерархии отделов

**Задание 2.2:** Выведите все отделы с указанием полного пути.

```sql
WITH RECURSIVE dept_tree AS (
    -- Базовая часть: отделы без родителя
    SELECT 
        dept_id,
        dept_name,
        parent_dept_id,
        1 AS level,
        dept_name::text AS full_path
    FROM departments
    WHERE parent_dept_id IS NULL
    
    UNION ALL
    
    -- Рекурсивная часть
    SELECT 
        d.dept_id,
        d.dept_name,
        d.parent_dept_id,
        dt.level + 1,
        dt.full_path || ' / ' || d.dept_name
    FROM departments d
    JOIN dept_tree dt ON d.parent_dept_id = dt.dept_id
)
SELECT 
    level,
    repeat('  ', level - 1) || dept_name AS tree_view,
    full_path
FROM dept_tree
ORDER BY full_path;
```

### 2.4. Генерация последовательностей

**Задание 2.3:** Сгенерируйте последовательность чисел от 1 до 10 и их квадраты.

```sql
WITH RECURSIVE numbers AS (
    SELECT 1 AS n
    UNION ALL
    SELECT n + 1 FROM numbers WHERE n < 10
)
SELECT n, n * n AS square FROM numbers;
```

**Задание 3.4:** Генерация всех дат за январь 2024.

```sql
WITH RECURSIVE dates AS (
    SELECT DATE '2024-01-01' AS d
    UNION ALL
    SELECT d + 1 FROM dates WHERE d < '2024-01-31'
)
SELECT d, to_char(d, 'Day') AS weekday FROM dates;
```

### 2.5. Защита от бесконечной рекурсии

**Задание 2.5:** Продемонстрируйте бесконечный цикл и способ его остановить.

```sql
-- ОПАСНО: этот запрос вызовет бесконечный цикл (не запускайте без LIMIT/WHERE)
-- WITH RECURSIVE infinite AS (
--     SELECT 1 AS n
--     UNION ALL
--     SELECT n + 1 FROM infinite
-- )
-- SELECT * FROM infinite;

-- Безопасный вариант с условием остановки:
WITH RECURSIVE safe AS (
    SELECT 1 AS n
    UNION ALL
    SELECT n + 1 FROM safe WHERE n < 100
)
SELECT * FROM safe;
```

**Также можно использовать `LIMIT`:**

```sql
WITH RECURSIVE infinite_limited AS (
    SELECT 1 AS n
    UNION ALL
    SELECT n + 1 FROM infinite_limited
)
SELECT * FROM infinite_limited LIMIT 10;
```

### 2.6. Поиск пути от сотрудника до CEO

**Задание 2.6:** Для каждого сотрудника выведите путь к CEO.

```sql
WITH RECURSIVE emp_path AS (
    -- Базовая: все сотрудники
    SELECT 
        emp_id,
        emp_name,
        manager_id,
        emp_name::text AS path,
        1 AS steps
    FROM employees
    
    UNION ALL
    
    -- Рекурсия: поднимаемся вверх по менеджерам
    SELECT 
        ep.emp_id,
        ep.emp_name,
        e.manager_id,
        ep.path || ' → ' || e.emp_name,
        ep.steps + 1
    FROM emp_path ep
    JOIN employees e ON ep.manager_id = e.emp_id
)
SELECT 
    emp_id,
    emp_name,
    path,
    steps
FROM emp_path
WHERE manager_id IS NULL
ORDER BY emp_id;
```


### 2.7. Практическое задание

**Задание 2.7:** Выведите для каждого отдела количество сотрудников **в нём и во всех его подчинённых отделах** (агрегация по дереву).

```sql
WITH RECURSIVE dept_descendants AS (
    -- Базовая: каждый отдел — свой собственный потомок
    SELECT 
        dept_id AS root_dept_id,
        dept_id AS child_dept_id,
        0 AS depth
    FROM departments
    
    UNION ALL
    
    -- Рекурсия: дочерние отделы
    SELECT 
        dd.root_dept_id,
        d.dept_id,
        dd.depth + 1
    FROM dept_descendants dd
    JOIN departments d ON d.parent_dept_id = dd.child_dept_id
)
SELECT 
    d.dept_name,
    COUNT(e.emp_id) AS total_emp_including_children
FROM dept_descendants dd
JOIN departments d ON d.dept_id = dd.root_dept_id
LEFT JOIN employees e ON e.dept_id = dd.child_dept_id
GROUP BY d.dept_id, d.dept_name
ORDER BY d.dept_name;
```


## Часть 4. Материализация CTE и оптимизация

### 4.1. Материализация по умолчанию

До PostgreSQL 12 CTE **всегда** материализовались — результат вычислялся один раз и сохранялся как временная таблица. Это могло быть как преимуществом (CTE используется несколько раз), так и недостатком (предикаты не «проталкивались» внутрь).

Начиная с PostgreSQL 12, CTE могут быть **встроены** (inlined), если они не содержат DML и используются только один раз.

### 4.2. Явное управление материализацией

**Задание 4.1:** Используйте `MATERIALIZED` и `NOT MATERIALIZED` для контроля.

```sql
-- MATERIALIZED: принудительная материализация
WITH dept_avg AS MATERIALIZED (
    SELECT dept_id, AVG(salary) AS avg_salary
    FROM employees
    GROUP BY dept_id
)
SELECT * FROM dept_avg WHERE dept_id = 1;

-- NOT MATERIALIZED: принудительное встраивание
WITH dept_avg AS NOT MATERIALIZED (
    SELECT dept_id, AVG(salary) AS avg_salary
    FROM employees
    GROUP BY dept_id
)
SELECT * FROM dept_avg WHERE dept_id = 1;
```

**Задание 4.2:** Сравните планы запросов.

```sql
-- План с MATERIALIZED
EXPLAIN (ANALYZE, COSTS OFF)
WITH dept_avg AS MATERIALIZED (
    SELECT dept_id, AVG(salary) AS avg_salary
    FROM employees
    GROUP BY dept_id
)
SELECT * FROM dept_avg WHERE dept_id = 1;

-- План с NOT MATERIALIZED
EXPLAIN (ANALYZE, COSTS OFF)
WITH dept_avg AS NOT MATERIALIZED (
    SELECT dept_id, AVG(salary) AS avg_salary
    FROM employees
    GROUP BY dept_id
)
SELECT * FROM dept_avg WHERE dept_id = 1;
```

**Что искать в плане:**
- `CTE Scan` — CTE материализован
- Подзапрос «встроен» прямо в основной план — CTE не материализован

### 4.3. Когда что использовать

| Ситуация | Рекомендация |
|----------|--------------|
| CTE используется несколько раз | `MATERIALIZED` (по умолчанию) |
| CTE содержит DML (`INSERT`/`UPDATE`/`DELETE`) | Автоматически `MATERIALIZED` |
| CTE используется один раз и содержит фильтр | `NOT MATERIALIZED` для оптимизации |
| CTE рекурсивный | Всегда материализуется |



## Очистка окружения

```sql
-- Удаление таблиц
DROP TABLE IF EXISTS big_data;
DROP TABLE IF EXISTS audit_log;
DROP TABLE IF EXISTS sales_archive;
DROP TABLE IF EXISTS sales;
DROP TABLE IF EXISTS employees CASCADE;
DROP TABLE IF EXISTS departments CASCADE;

-- Переключение и удаление базы
\c postgres
DROP DATABASE IF EXISTS lab_cte;
```

**Успешной работы!**
