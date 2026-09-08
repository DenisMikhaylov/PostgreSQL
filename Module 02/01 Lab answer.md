
### 1.2. Проектирование нормализованной схемы

**Задание 1.2:** Разработайте нормализованную схему базы данных (3НФ) для интернет-магазина. Она должна включать таблицы для клиентов, заказов, товаров и позиций заказа.


```sql
-- ТАБЛИЦА КЛИЕНТОВ
CREATE TABLE customers (
    customer_id SERIAL PRIMARY KEY,
    name VARCHAR(100) NOT NULL,
    phone VARCHAR(20) NOT NULL,
    email VARCHAR(100) UNIQUE,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);

-- ТАБЛИЦА ТОВАРОВ
CREATE TABLE products (
    product_id SERIAL PRIMARY KEY,
    name VARCHAR(200) NOT NULL,
    description TEXT,
    price DECIMAL(10,2) NOT NULL CHECK (price >= 0),
    stock_quantity INTEGER DEFAULT 0 CHECK (stock_quantity >= 0)
);

-- ТАБЛИЦА ЗАКАЗОВ
CREATE TABLE orders (
    order_id SERIAL PRIMARY KEY,
    customer_id INTEGER NOT NULL REFERENCES customers(customer_id),
    order_date TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    status VARCHAR(20) DEFAULT 'new' CHECK (status IN ('new', 'processing', 'shipped', 'delivered', 'cancelled'))
);

-- ТАБЛИЦА ПОЗИЦИЙ ЗАКАЗА (связь многие-ко-многим)
CREATE TABLE order_items (
    order_id INTEGER NOT NULL REFERENCES orders(order_id) ON DELETE CASCADE,
    product_id INTEGER NOT NULL REFERENCES products(product_id),
    quantity INTEGER NOT NULL CHECK (quantity > 0),
    price_at_time DECIMAL(10,2) NOT NULL CHECK (price_at_time >= 0),
    PRIMARY KEY (order_id, product_id)
);
```
