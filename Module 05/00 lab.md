# Лабораторная работа: Соединения, множества, представления и оконные функции


**Цель работы:** Изучить на практике все виды соединений таблиц (INNER, LEFT, RIGHT, FULL, CROSS, NATURAL, LATERAL), операции над множествами (UNION, UNION ALL, INTERSECT, EXCEPT), создание обычных и материализованных представлений, а также оконные функции (SUM, LAG, LEAD) с использованием PARTITION BY и ORDER BY.


## Часть 0. Подготовка окружения

### 0.1. Создание базы данных

```sql
CREATE DATABASE lab_joins;

\c lab_joins
```

### 0.2. Создание тестовых таблиц

```sql
-- Таблица отделов
CREATE TABLE departments (
    dept_id   INTEGER PRIMARY KEY,
    dept_name TEXT NOT NULL,
    city      TEXT
);

-- Таблица сотрудников
CREATE TABLE employees (
    emp_id    INTEGER PRIMARY KEY,
    emp_name  TEXT NOT NULL,
    dept_id   INTEGER,
    position  TEXT,
    salary    NUMERIC(10,2),
    hired_at  DATE
);

-- Таблица проектов
CREATE TABLE projects (
    project_id   INTEGER PRIMARY KEY,
    project_name TEXT NOT NULL,
    dept_id      INTEGER,
    budget       NUMERIC(12,2)
);

-- Таблица назначений сотрудников на проекты (связь многие-ко-многим)
CREATE TABLE assignments (
    emp_id     INTEGER,
    project_id INTEGER,
    hours      INTEGER,
    PRIMARY KEY (emp_id, project_id)
);
```

### 0.3. Заполнение тестовыми данными

```sql
-- Отделы (обратите внимание: у отдела 4 нет сотрудников, а у сотрудника 5 нет отдела)
INSERT INTO departments (dept_id, dept_name, city) VALUES
    (1, 'Разработка',   'Москва'),
    (2, 'Аналитика',    'Санкт-Петербург'),
    (3, 'Маркетинг',    'Казань'),
    (4, 'Безопасность', 'Новосибирск');

-- Сотрудники
INSERT INTO employees (emp_id, emp_name, dept_id, position, salary, hired_at) VALUES
    (101, 'Иванов',   1, 'Senior Developer', 180000, '2019-03-15'),
    (102, 'Петров',   1, 'Developer',        120000, '2020-07-01'),
    (103, 'Сидоров',  1, 'Junior Developer',  80000, '2022-09-10'),
    (104, 'Козлов',   2, 'Data Analyst',     130000, '2020-02-20'),
    (105, 'Морозов',  2, 'BI Analyst',       110000, '2021-06-15'),
    (106, 'Новиков',  3, 'Marketer',          95000, '2021-11-05'),
    (107, 'Орлов',    3, 'CMO',              160000, '2018-01-10'),
    (108, 'Волков',  NULL, 'Consultant',      70000, '2023-04-01');

-- Проекты
INSERT INTO projects (project_id, project_name, dept_id, budget) VALUES
    (201, 'CRM System',        1, 5000000),
    (202, 'Data Warehouse',    2, 3000000),
    (203, 'Marketing Campaign',3, 1500000),
    (204, 'AI Platform',       1, 8000000),
    (205, 'Audit System',      4, 2000000);

-- Назначения на проекты
INSERT INTO assignments (emp_id, project_id, hours) VALUES
    (101, 201, 120),
    (101, 204, 200),
    (102, 201, 300),
    (103, 201,  80),
    (104, 202, 250),
    (105, 202, 150),
    (106, 203, 180),
    (107, 203,  60),
    (101, 202,  40);
```

### 0.4. Проверка данных

```sql
SELECT * FROM departments;
SELECT * FROM employees;
SELECT * FROM projects;
SELECT * FROM assignments;
```


## Часть 1. Соединения таблиц (JOIN)

### 1.1. INNER JOIN — внутреннее соединение

**INNER JOIN** возвращает только те строки, для которых нашлось соответствие в обеих таблицах.

**Задание 1.1:** Выведите список сотрудников с названиями их отделов.

```sql
SELECT 
    e.emp_id,
    e.emp_name,
    e.position,
    d.dept_name
FROM employees e
INNER JOIN departments d ON e.dept_id = d.dept_id
ORDER BY e.emp_id;
```

**Обратите внимание:** сотрудник `Волков` (dept_id = NULL) **не попал** в результат, так как у него нет отдела. Отдел `Безопасность` тоже не попал — в нём нет сотрудников.

**Задание 1.2:** Выведите список сотрудников с их проектами и количеством часов.

```sql
SELECT 
    e.emp_name,
    p.project_name,
    a.hours
FROM employees e
INNER JOIN assignments a ON e.emp_id = a.emp_id
INNER JOIN projects p ON a.project_id = p.project_id
ORDER BY e.emp_name, p.project_name;
```

### 1.2. LEFT JOIN (LEFT OUTER JOIN) — левое внешнее соединение

**LEFT JOIN** возвращает **все строки из левой таблицы**, а для строк без соответствия в правой таблице подставляет `NULL`.

**Задание 1.3:** Выведите всех сотрудников с названиями их отделов (включая тех, у кого нет отдела).

```sql
SELECT 
    e.emp_id,
    e.emp_name,
    e.position,
    d.dept_name
FROM employees e
LEFT JOIN departments d ON e.dept_id = d.dept_id
ORDER BY e.emp_id;
```

**Результат:** теперь в списке появится `Волков` с `dept_name = NULL`.

**Задание 1.4:** Найдите сотрудников без отдела (анти-соединение).

```sql
SELECT e.emp_id, e.emp_name
FROM employees e
LEFT JOIN departments d ON e.dept_id = d.dept_id
WHERE d.dept_id IS NULL;
```

**Задание 1.5:** Выведите все отделы и количество сотрудников в каждом (включая пустые отделы).

```sql
SELECT 
    d.dept_name,
    COUNT(e.emp_id) AS employee_count
FROM departments d
LEFT JOIN employees e ON d.dept_id = e.dept_id
GROUP BY d.dept_id, d.dept_name
ORDER BY d.dept_name;
```

**Результат:** отдел `Безопасность` покажет `0` (а не отсутствие строки).

### 1.3. RIGHT JOIN (RIGHT OUTER JOIN) — правое внешнее соединение

**RIGHT JOIN** — зеркальное отражение `LEFT JOIN`: возвращает все строки из **правой** таблицы.

**Задание 1.6:** Перепишите запрос 1.3 через RIGHT JOIN.

```sql
SELECT 
    e.emp_id,
    e.emp_name,
    d.dept_name
FROM departments d
RIGHT JOIN employees e ON e.dept_id = d.dept_id
ORDER BY e.emp_id;
```

**Задание 1.7:** Выведите все отделы, даже те, где нет проектов.

```sql
SELECT 
    d.dept_name,
    p.project_name
FROM projects p
RIGHT JOIN departments d ON p.dept_id = d.dept_id
ORDER BY d.dept_name;
```

> **Совет:** `RIGHT JOIN` можно всегда заменить на `LEFT JOIN`, поменяв таблицы местами. В реальных проектах чаще используют `LEFT JOIN` как более читаемый.

### 1.4. FULL JOIN (FULL OUTER JOIN) — полное внешнее соединение

**FULL JOIN** возвращает **все строки из обеих таблиц**. Если для строки нет соответствия в одной из таблиц, для её столбцов подставляется `NULL`.

**Задание 1.8:** Полное соединение сотрудников и отделов.

```sql
SELECT 
    e.emp_id,
    e.emp_name,
    d.dept_id,
    d.dept_name
FROM employees e
FULL JOIN departments d ON e.dept_id = d.dept_id
ORDER BY e.emp_id NULLS LAST, d.dept_id;
```

**Результат:** будут и `Волков` (без отдела), и отдел `Безопасность` (без сотрудников).

**Задание 1.9:** Симметричная разница — что есть в одной таблице, но нет в другой.

```sql
SELECT 
    e.emp_id,
    e.emp_name,
    d.dept_id,
    d.dept_name
FROM employees e
FULL JOIN departments d ON e.dept_id = d.dept_id
WHERE e.emp_id IS NULL OR d.dept_id IS NULL;
```

Это аналог `EXCEPT` в обе стороны.

### 1.5. CROSS JOIN — декартово произведение

**CROSS JOIN** возвращает **все возможные пары строк** из двух таблиц. Условие соединения отсутствует.

**Задание 1.10:** Все возможные комбинации отделов и проектов.

```sql
SELECT 
    d.dept_name,
    p.project_name
FROM departments d
CROSS JOIN projects p
ORDER BY d.dept_name, p.project_name;
```

**Результат:** `4 отдела × 5 проектов = 20 строк`.

> **Осторожно:** `CROSS JOIN` на больших таблицах очень быстро приводит к огромным результатам. Например, `10000 × 10000 = 100 000 000` строк.

**Задание 1.11:** Сгенерируйте все пары сотрудников одного отдела (для дальнейшего сравнения).

```sql
SELECT 
    e1.emp_name AS emp1,
    e2.emp_name AS emp2,
    e1.dept_id
FROM employees e1
CROSS JOIN employees e2
WHERE e1.dept_id = e2.dept_id
  AND e1.emp_id < e2.emp_id
ORDER BY e1.dept_id;
```

### 1.6. NATURAL JOIN — естественное соединение

**NATURAL JOIN** автоматически соединяет таблицы по **всем столбцам с одинаковыми именами**. Общие столбцы в результате выводятся один раз.

**Задание 1.12:** Естественное соединение сотрудников и отделов.

```sql
SELECT *
FROM employees
NATURAL JOIN departments;
```

**Результат:** соединение по столбцу `dept_id`. В выводе `dept_id` будет один раз.

**Эквивалент:**
```sql
SELECT *
FROM employees
JOIN departments USING (dept_id);
```

**Задание 1.13:** Продемонстрируйте **опасность NATURAL JOIN** — создайте таблицу с неожиданным одноимённым столбцом.

```sql
-- Создаём таблицу с колонкой "city", которая есть и в departments
CREATE TABLE offices (
    office_id INTEGER PRIMARY KEY,
    dept_id   INTEGER,
    city      TEXT
);

INSERT INTO offices VALUES
    (1, 1, 'Москва'),
    (2, 2, 'Москва'),       -- обратите внимание: другой город, но совпадает
    (3, 3, 'Казань');

-- NATURAL JOIN соединит и по dept_id, и по city!
SELECT *
FROM offices
NATURAL JOIN departments;
```

**Результат:** строка `(2, 2, 'Москва')` **не соединится** с отделом 2 (`'Санкт-Петербург'`), потому что `city` не совпадает. Это классический пример «тихой» ошибки `NATURAL JOIN`.

> **Вывод:** в production используйте `JOIN ... USING (...)` или `JOIN ... ON ...` — они явные и безопасные.

### 1.7. LATERAL JOIN — боковое соединение

**LATERAL JOIN** позволяет подзапросу или функции в правой части ссылаться на столбцы **левой** таблицы.

**Задание 1.14:** Для каждого отдела выведите топ-2 самых высокооплачиваемых сотрудника.

```sql
SELECT 
    d.dept_name,
    e.emp_name,
    e.salary
FROM departments d
CROSS JOIN LATERAL (
    SELECT emp_name, salary
    FROM employees
    WHERE dept_id = d.dept_id
    ORDER BY salary DESC
    LIMIT 2
) e
ORDER BY d.dept_name, e.salary DESC;
```

**Задание 1.15:** LEFT JOIN LATERAL — сохраняет отделы без сотрудников.

```sql
SELECT 
    d.dept_name,
    e.emp_name,
    e.salary
FROM departments d
LEFT JOIN LATERAL (
    SELECT emp_name, salary
    FROM employees
    WHERE dept_id = d.dept_id
    ORDER BY salary DESC
    LIMIT 1
) e ON true
ORDER BY d.dept_name;
```

**Результат:** отдел `Безопасность` появится с `emp_name = NULL`.

**Задание 1.16:** LATERAL с функцией, возвращающей множество — «разверните» проекты по кварталам.

```sql
SELECT 
    p.project_name,
    p.budget,
    q.qtr,
    p.budget / 4 AS quarterly_budget
FROM projects p
CROSS JOIN LATERAL generate_series(1, 4) AS q(qtr)
ORDER BY p.project_name, q.qtr;
```

### 1.8. Сводная таблица JOIN

| Тип | Что вернёт |
|-----|------------|
| `INNER JOIN` | Только совпадающие строки |
| `LEFT JOIN` | Все строки слева + совпадения справа |
| `RIGHT JOIN` | Все строки справа + совпадения слева |
| `FULL JOIN` | Все строки из обеих таблиц |
| `CROSS JOIN` | Декартово произведение |
| `NATURAL JOIN` | Автоматически по всем одноимённым столбцам |
| `LATERAL` | Модификатор для ссылки на левую таблицу |


## Часть 2. Операции над множествами

Операции над множествами объединяют результаты **двух запросов** с одинаковым числом и типами столбцов.

### 2.1. UNION — объединение с удалением дубликатов

**UNION** объединяет результаты двух запросов и **удаляет дубликаты**.

**Задание 2.1:** Список всех городов, где есть отделы или офисы.

```sql
SELECT city FROM departments
UNION
SELECT city FROM offices
ORDER BY city;
```

**Результат:** `Москва`, `Санкт-Петербург`, `Казань`, `Новосибирск` — каждый город один раз.

### 2.2. UNION ALL — объединение с сохранением дубликатов

**UNION ALL** объединяет результаты **без удаления** дубликатов и работает быстрее.

**Задание 2.2:** То же самое, но с сохранением дубликатов.

```sql
SELECT city FROM departments
UNION ALL
SELECT city FROM offices
ORDER BY city;
```

**Результат:** `Москва` появится **дважды**.

> **Совет:** если вы уверены, что дубликатов нет (или они допустимы) — всегда используйте `UNION ALL`, он не тратит ресурсы на сортировку и удаление дублей.

### 2.3. INTERSECT — пересечение

**INTERSECT** возвращает только те строки, которые есть **в обоих** результатах.

**Задание 2.3:** Города, где есть **и отдел, и офис**.

```sql
SELECT city FROM departments
INTERSECT
SELECT city FROM offices
ORDER BY city;
```

**Результат:** `Москва`, `Казань` (Санкт-Петербург есть только в отделах, Новосибирск — только в отделах).

### 2.4. EXCEPT — разность

**EXCEPT** возвращает строки из первого результата, которых **нет** во втором.

**Задание 2.4:** Города, где есть отдел, но **нет** офиса.

```sql
SELECT city FROM departments
EXCEPT
SELECT city FROM offices
ORDER BY city;
```

**Результат:** `Санкт-Петербург`, `Новосибирск`.

**Задание 2.5:** Симметричная разница (в одну и в другую сторону).

```sql
(SELECT city FROM departments EXCEPT SELECT city FROM offices)
UNION
(SELECT city FROM offices EXCEPT SELECT city FROM departments)
ORDER BY city;
```

### 2.5. Сводная таблица операций

| Операция | Что возвращает | Дубликаты |
|----------|----------------|-----------|
| `UNION` | Всё из A и B | Удаляются |
| `UNION ALL` | Всё из A и B | Сохраняются |
| `INTERSECT` | Только то, что есть в A и B | Удаляются |
| `EXCEPT` | То, что есть в A, но нет в B | Удаляются |

> **Важно:** все операции требуют **одинакового количества столбцов** и **совместимых типов**.

### 2.6. Практическое задание

**Задание 2.6:** Выведите список всех ID сотрудников, которые либо работают в отделе 1, либо назначены на проект 202 (объединение).

```sql
SELECT emp_id FROM employees WHERE dept_id = 1
UNION
SELECT emp_id FROM assignments WHERE project_id = 202
ORDER BY emp_id;
```


## Часть 3. Представления (VIEW) и материализованные представления

### 3.1. Обычное представление (VIEW)

**VIEW** — это виртуальная таблица, которая не хранит данные, а выполняет запрос при каждом обращении.

**Задание 3.1:** Создайте представление с информацией о сотрудниках и их отделах.

```sql
CREATE VIEW v_employee_departments AS
SELECT 
    e.emp_id,
    e.emp_name,
    e.position,
    e.salary,
    d.dept_name,
    d.city
FROM employees e
LEFT JOIN departments d ON e.dept_id = d.dept_id;
```

**Задание 3.2:** Используйте представление как обычную таблицу.

```sql
-- Выборка из представления
SELECT * FROM v_employee_departments;

-- Фильтрация
SELECT * FROM v_employee_departments WHERE city = 'Москва';

-- Агрегация
SELECT dept_name, count(*), avg(salary)
FROM v_employee_departments
GROUP BY dept_name;
```

**Задание 3.3:** Создайте представление с агрегацией по отделам.

```sql
CREATE VIEW v_dept_stats AS
SELECT 
    d.dept_id,
    d.dept_name,
    COUNT(e.emp_id)         AS employee_count,
    COALESCE(AVG(e.salary), 0) AS avg_salary,
    COALESCE(SUM(e.salary), 0) AS total_salary,
    COALESCE(MAX(e.salary), 0) AS max_salary
FROM departments d
LEFT JOIN employees e ON d.dept_id = e.dept_id
GROUP BY d.dept_id, d.dept_name;

SELECT * FROM v_dept_stats ORDER BY dept_name;
```

**Задание 3.4:** Обновление данных через представление (если возможно).

```sql
-- Простое представление без JOIN — обновляемо
CREATE VIEW v_high_salary AS
SELECT emp_id, emp_name, salary FROM employees WHERE salary > 100000;

-- Обновление через представление
UPDATE v_high_salary SET salary = salary + 10000 WHERE emp_id = 101;

-- Проверка
SELECT * FROM employees WHERE emp_id = 101;
```

> **Правило:** представление обновляемо, если оно основано на одной таблице и не содержит `DISTINCT`, `GROUP BY`, `UNION` и т.п.

**Задание 3.5:** Удаление представления.

```sql
DROP VIEW v_high_salary;
```

### 3.2. Материализованное представление (MATERIALIZED VIEW)

**MATERIALIZED VIEW** **физически хранит** результат запроса. Данные не обновляются автоматически — нужно вызывать `REFRESH`.

**Задание 3.6:** Создайте материализованное представление с топ-3 проекта по бюджету в каждом отделе.

```sql
CREATE MATERIALIZED VIEW mv_top_projects AS
SELECT 
    d.dept_name,
    p.project_name,
    p.budget,
    RANK() OVER (PARTITION BY d.dept_id ORDER BY p.budget DESC) AS rank_in_dept
FROM projects p
JOIN departments d ON p.dept_id = d.dept_id
WHERE p.budget > 0;

SELECT * FROM mv_top_projects ORDER BY dept_name, rank_in_dept;
```

**Задание 3.7:** Создайте материализованное представление с индексом для ускорения запросов.

```sql
CREATE MATERIALIZED VIEW mv_dept_salary AS
SELECT 
    d.dept_id,
    d.dept_name,
    COUNT(e.emp_id)          AS employee_count,
    AVG(e.salary)            AS avg_salary,
    SUM(e.salary)            AS total_salary
FROM departments d
LEFT JOIN employees e ON d.dept_id = e.dept_id
GROUP BY d.dept_id, d.dept_name;

-- Создаём индекс
CREATE UNIQUE INDEX idx_mv_dept_salary ON mv_dept_salary(dept_id);

-- Запрос с индексом
SELECT * FROM mv_dept_salary WHERE dept_id = 1;
```

**Задание 3.8:** Измените данные в исходных таблицах и убедитесь, что MV не обновилось автоматически.

```sql
-- Добавляем сотрудника в отдел 1
INSERT INTO employees (emp_id, emp_name, dept_id, position, salary, hired_at)
VALUES (109, 'Лебедев', 1, 'Architect', 250000, '2017-05-01');

-- Проверяем MV — данные ещё старые
SELECT * FROM mv_dept_salary WHERE dept_id = 1;

-- Проверяем обычное представление — данные сразу обновились
SELECT * FROM v_dept_stats WHERE dept_id = 1;
```

**Задание 3.9:** Обновите материализованное представление.

```sql
-- Полное обновление (по умолчанию блокирует читателей)
REFRESH MATERIALIZED VIEW mv_dept_salary;

-- Проверка — теперь данные актуальны
SELECT * FROM mv_dept_salary WHERE dept_id = 1;
```

**Задание 3.10:** Обновление без блокировки (Concurrent).

```sql
-- Требуется UNIQUE индекс (мы его создали)
REFRESH MATERIALIZED VIEW CONCURRENTLY mv_dept_salary;
```

> **Важно:** `CONCURRENTLY` работает только при наличии `UNIQUE` индекса и не блокирует читателей.



## Часть 4. Оконные функции (OVER, PARTITION BY, ORDER BY) 

### 4.1. Основы оконных функций

**Оконная функция** выполняет вычисления над набором строк, связанных с текущей строкой, **не сворачивая** результат в одну строку (в отличие от `GROUP BY`).

Общий синтаксис:

```sql
функция() OVER (
    [PARTITION BY ...]
    [ORDER BY ...]
    [ROWS/RANGE ...]
)
```

### 4.2. SUM() OVER — накопительная сумма

**Задание 4.1:** Для каждого сотрудника выведите зарплату и общую сумму зарплат в его отделе.

```sql
SELECT 
    e.emp_name,
    d.dept_name,
    e.salary,
    SUM(e.salary) OVER (PARTITION BY e.dept_id) AS total_dept_salary
FROM employees e
LEFT JOIN departments d ON e.dept_id = d.dept_id
ORDER BY d.dept_name, e.emp_name;
```

**Задание 4.2:** Накопительная сумма зарплат при сортировке по дате найма.

```sql
SELECT 
    e.emp_name,
    e.hired_at,
    e.salary,
    SUM(e.salary) OVER (ORDER BY e.hired_at) AS running_total
FROM employees e
WHERE e.dept_id = 1
ORDER BY e.hired_at;
```

**Задание 4.3:** Накопительная сумма **внутри** каждого отдела.

```sql
SELECT 
    e.emp_name,
    d.dept_name,
    e.salary,
    SUM(e.salary) OVER (
        PARTITION BY e.dept_id 
        ORDER BY e.hired_at
    ) AS running_total_in_dept
FROM employees e
JOIN departments d ON e.dept_id = d.dept_id
ORDER BY d.dept_name, e.hired_at;
```

**Задание 4.4:** Процент от общей суммы по отделу.

```sql
SELECT 
    e.emp_name,
    d.dept_name,
    e.salary,
    ROUND(100.0 * e.salary / SUM(e.salary) OVER (PARTITION BY e.dept_id), 2) AS pct_of_dept
FROM employees e
JOIN departments d ON e.dept_id = d.dept_id
ORDER BY d.dept_name, pct_of_dept DESC;
```

### 4.3. LAG() OVER — значение предыдущей строки

**LAG()** возвращает значение из строки, находящейся на N позиций **раньше** текущей.

**Синтаксис:**
```sql
LAG(column, offset, default_value) OVER (ORDER BY ...)
```

**Задание 4.5:** Для каждого сотрудника выведите его зарплату и зарплату предыдущего (по дате найма).

```sql
SELECT 
    e.emp_name,
    e.hired_at,
    e.salary,
    LAG(e.salary) OVER (ORDER BY e.hired_at) AS prev_salary,
    e.salary - LAG(e.salary) OVER (ORDER BY e.hired_at) AS salary_diff
FROM employees e
WHERE e.dept_id = 1
ORDER BY e.hired_at;
```

**Задание 4.6:** LAG с указанием смещения и значения по умолчанию.

```sql
SELECT 
    e.emp_name,
    e.hired_at,
    e.salary,
    LAG(e.salary, 2, 0) OVER (ORDER BY e.hired_at) AS salary_2_positions_back
FROM employees e
WHERE e.dept_id = 1
ORDER BY e.hired_at;
```

Здесь `2` — смещение (2 позиции назад), `0` — значение по умолчанию для первых строк.

**Задание 4.7:** LAG внутри каждого отдела (PARTITION BY).

```sql
SELECT 
    e.emp_name,
    d.dept_name,
    e.hired_at,
    e.salary,
    LAG(e.salary) OVER (
        PARTITION BY e.dept_id 
        ORDER BY e.hired_at
    ) AS prev_salary_in_dept
FROM employees e
JOIN departments d ON e.dept_id = d.dept_id
ORDER BY d.dept_name, e.hired_at;
```

### 4.4. LEAD() OVER — значение следующей строки

**LEAD()** — зеркальное отражение `LAG()`: возвращает значение из строки **после** текущей.

**Задание 4.8:** Для каждого сотрудника выведите его зарплату и зарплату следующего по дате найма.

```sql
SELECT 
    e.emp_name,
    e.hired_at,
    e.salary,
    LEAD(e.salary) OVER (ORDER BY e.hired_at) AS next_salary,
    LEAD(e.emp_name) OVER (ORDER BY e.hired_at) AS next_employee
FROM employees e
WHERE e.dept_id = 1
ORDER BY e.hired_at;
```

**Задание 4.9:** LAG и LEAD одновременно — «соседи» по отделу.

```sql
SELECT 
    e.emp_name,
    d.dept_name,
    e.salary,
    LAG(e.emp_name)  OVER w AS prev_employee,
    LEAD(e.emp_name) OVER w AS next_employee
FROM employees e
JOIN departments d ON e.dept_id = d.dept_id
WINDOW w AS (PARTITION BY e.dept_id ORDER BY e.salary DESC)
ORDER BY d.dept_name, e.salary DESC;
```

> **Обратите внимание:** здесь используется конструкция `WINDOW w AS (...)` — это именованное окно, позволяющее не повторять одно и то же определение.

### 4.5. Дополнительные оконные функции

**Задание 4.10:** Использование `ROW_NUMBER()`, `RANK()`, `DENSE_RANK()`.

```sql
SELECT 
    e.emp_name,
    d.dept_name,
    e.salary,
    ROW_NUMBER() OVER (PARTITION BY e.dept_id ORDER BY e.salary DESC) AS row_num,
    RANK()       OVER (PARTITION BY e.dept_id ORDER BY e.salary DESC) AS rank,
    DENSE_RANK() OVER (PARTITION BY e.dept_id ORDER BY e.salary DESC) AS dense_rank
FROM employees e
JOIN departments d ON e.dept_id = d.dept_id
ORDER BY d.dept_name, e.salary DESC;
```

**Разница:**
- `ROW_NUMBER` — всегда уникальный номер (1, 2, 3, 4...)
- `RANK` — при равенстве одинаковый ранг, но следующий номер «перепрыгивает» (1, 2, 2, 4...)
- `DENSE_RANK` — при равенстве одинаковый ранг, но следующий идёт по порядку (1, 2, 2, 3...)

**Задание 4.11:** Топ-1 сотрудник в каждом отделе через оконную функцию.

```sql
WITH ranked AS (
    SELECT 
        e.emp_name,
        d.dept_name,
        e.salary,
        ROW_NUMBER() OVER (PARTITION BY e.dept_id ORDER BY e.salary DESC) AS rn
    FROM employees e
    JOIN departments d ON e.dept_id = d.dept_id
)
SELECT dept_name, emp_name, salary
FROM ranked
WHERE rn = 1
ORDER BY dept_name;
```

### 4.6. Практическое задание: анализ динамики

**Задание 4.12:** Для каждого проекта выведите часы по сотрудникам и разницу с предыдущим сотрудником.

```sql
SELECT 
    p.project_name,
    e.emp_name,
    a.hours,
    LAG(a.hours) OVER (PARTITION BY a.project_id ORDER BY a.hours DESC) AS prev_hours,
    LEAD(a.hours) OVER (PARTITION BY a.project_id ORDER BY a.hours DESC) AS next_hours,
    SUM(a.hours) OVER (PARTITION BY a.project_id) AS total_project_hours
FROM assignments a
JOIN employees e ON a.emp_id = e.emp_id
JOIN projects p ON a.project_id = p.project_id
ORDER BY p.project_name, a.hours DESC;
```

**Задание 4.13:** Доля сотрудника в общем количестве часов по проекту.

```sql
SELECT 
    p.project_name,
    e.emp_name,
    a.hours,
    SUM(a.hours) OVER (PARTITION BY a.project_id) AS total_hours,
    ROUND(100.0 * a.hours / SUM(a.hours) OVER (PARTITION BY a.project_id), 2) AS pct
FROM assignments a
JOIN employees e ON a.emp_id = e.emp_id
JOIN projects p ON a.project_id = p.project_id
ORDER BY p.project_name, pct DESC;
```


## Очистка окружения

```sql
-- Удаление представлений
DROP MATERIALIZED VIEW IF EXISTS mv_employee_analytics;
DROP MATERIALIZED VIEW IF EXISTS mv_dept_salary;
DROP MATERIALIZED VIEW IF EXISTS mv_top_projects;
DROP VIEW IF EXISTS v_dept_stats;
DROP VIEW IF EXISTS v_employee_departments;
DROP VIEW IF EXISTS v_high_salary;

-- Удаление таблиц
DROP TABLE IF EXISTS assignments CASCADE;
DROP TABLE IF EXISTS projects CASCADE;
DROP TABLE IF EXISTS employees CASCADE;
DROP TABLE IF EXISTS departments CASCADE;
DROP TABLE IF EXISTS offices CASCADE;

-- Переключение и удаление базы
\c postgres
DROP DATABASE IF EXISTS lab_joins;
```

**Успешной работы!**
