# جزوه جامع PostgreSQL (با مقایسه MS SQL)

## فصل 1: آشنایی اولیه

### 1.1 PostgreSQL چیست؟
PostgreSQL یک سیستم مدیریت پایگاه داده رابطه‌ای (RDBMS) متن‌باز و قدرتمند است که از SQL استاندارد پیروی می‌کند.

**تفاوت با MS SQL:**
- PostgreSQL متن‌باز و رایگان است
- MS SQL محصول مایکروسافت است
- PostgreSQL بر روی Linux, Windows, MacOS اجرا می‌شود
- PostgreSQL به استانداردهای SQL نزدیک‌تر است

### 1.2 نصب و راه‌اندازی

**ابزارهای مورد نیاز:**
- PostgreSQL Server
- pgAdmin (معادل SQL Server Management Studio)
- یا psql (Command Line Tool)

### 1.3 اتصال به دیتابیس

```sql
-- در psql
psql -U username -d database_name

-- MS SQL معادل:
-- sqlcmd -S server -d database -U username
```

---

## فصل 2: مفاهیم پایه

### 2.1 ساختار کلی

```
Server
  └─ Database (دیتابیس)
      └─ Schema (طرحواره)
          └─ Table (جدول)
```

**تفاوت مهم:** در PostgreSQL مفهوم Schema بسیار مهم‌تر است. Schema پیش‌فرض `public` است.

```sql
-- PostgreSQL
CREATE SCHEMA my_schema;
CREATE TABLE my_schema.users (...);

-- MS SQL
-- همین کار رو با دیتابیس‌های جداگانه انجام می‌دادی
```

### 2.2 انواع داده‌ها (Data Types)

| PostgreSQL | MS SQL | توضیح |
|-----------|--------|-------|
| `SERIAL` | `IDENTITY` | شماره خودکار |
| `VARCHAR(n)` | `VARCHAR(n)` | متن با طول متغیر |
| `TEXT` | `VARCHAR(MAX)` | متن بدون محدودیت |
| `INTEGER` | `INT` | عدد صحیح |
| `NUMERIC(p,s)` | `DECIMAL(p,s)` | اعشاری دقیق |
| `BOOLEAN` | `BIT` | بولین |
| `TIMESTAMP` | `DATETIME` | تاریخ و زمان |
| `DATE` | `DATE` | فقط تاریخ |
| `JSON/JSONB` | `NVARCHAR(MAX)` | جیسون |
| `ARRAY` | - | آرایه (ندارد در SQL Server) |

### 2.3 ایجاد دیتابیس

```sql
-- PostgreSQL
CREATE DATABASE my_database
    ENCODING = 'UTF8'
    LC_COLLATE = 'en_US.UTF-8'
    LC_CTYPE = 'en_US.UTF-8';

-- MS SQL معادل:
-- CREATE DATABASE my_database
-- COLLATE Persian_100_CI_AS;
```

---

## فصل 3: جداول (Tables)

### 3.1 ایجاد جدول

```sql
-- PostgreSQL
CREATE TABLE users (
    id SERIAL PRIMARY KEY,
    username VARCHAR(50) NOT NULL UNIQUE,
    email VARCHAR(100) NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    is_active BOOLEAN DEFAULT TRUE
);

-- MS SQL معادل:
-- CREATE TABLE users (
--     id INT IDENTITY(1,1) PRIMARY KEY,
--     username VARCHAR(50) NOT NULL UNIQUE,
--     email VARCHAR(100) NOT NULL,
--     created_at DATETIME DEFAULT GETDATE(),
--     is_active BIT DEFAULT 1
-- );
```

### 3.2 کلیدهای خارجی (Foreign Keys)

```sql
CREATE TABLE posts (
    id SERIAL PRIMARY KEY,
    user_id INTEGER REFERENCES users(id) ON DELETE CASCADE,
    title VARCHAR(200) NOT NULL,
    content TEXT,
    created_at TIMESTAMP DEFAULT NOW()
);

-- CASCADE گزینه‌های:
-- ON DELETE CASCADE: با حذف user، پست‌هاش هم پاک می‌شود
-- ON DELETE SET NULL: با حذف user، user_id برابر NULL می‌شود
-- ON DELETE RESTRICT: اجازه حذف user را نمی‌دهد
```

### 3.3 ویرایش جدول

```sql
-- اضافه کردن ستون
ALTER TABLE users ADD COLUMN phone VARCHAR(15);

-- حذف ستون
ALTER TABLE users DROP COLUMN phone;

-- تغییر نوع داده
ALTER TABLE users ALTER COLUMN email TYPE VARCHAR(150);

-- اضافه کردن constraint
ALTER TABLE users ADD CONSTRAINT email_check CHECK (email LIKE '%@%');
```

---

## فصل 4: کوئری‌های پایه (Basic Queries)

### 4.1 SELECT

```sql
-- ساده
SELECT * FROM users;

-- با شرط
SELECT username, email FROM users WHERE is_active = TRUE;

-- با مرتب‌سازی
SELECT * FROM users ORDER BY created_at DESC;

-- با محدودیت تعداد
SELECT * FROM users LIMIT 10 OFFSET 5;

-- MS SQL معادل LIMIT:
-- SELECT TOP 10 * FROM users;
-- یا در نسخه‌های جدید:
-- SELECT * FROM users ORDER BY id OFFSET 5 ROWS FETCH NEXT 10 ROWS ONLY;
```

### 4.2 WHERE شرط‌ها

```sql
-- عملگرهای مقایسه
SELECT * FROM users WHERE id > 5;
SELECT * FROM users WHERE username = 'john';

-- LIKE برای جستجوی متن
SELECT * FROM users WHERE email LIKE '%@gmail.com';

-- IN برای چند مقدار
SELECT * FROM users WHERE id IN (1, 3, 5, 7);

-- BETWEEN برای بازه
SELECT * FROM posts WHERE created_at BETWEEN '2024-01-01' AND '2024-12-31';

-- IS NULL
SELECT * FROM users WHERE phone IS NULL;

-- عملگرهای منطقی
SELECT * FROM users WHERE is_active = TRUE AND email LIKE '%@gmail.com';
```

### 4.3 INSERT

```sql
-- تک رکورد
INSERT INTO users (username, email) 
VALUES ('john_doe', 'john@example.com');

-- چند رکورد
INSERT INTO users (username, email) 
VALUES 
    ('alice', 'alice@example.com'),
    ('bob', 'bob@example.com');

-- دریافت ID اضافه شده
INSERT INTO users (username, email) 
VALUES ('charlie', 'charlie@example.com')
RETURNING id, username;

-- MS SQL معادل RETURNING:
-- بعد از INSERT از SCOPE_IDENTITY() استفاده می‌کردی
```

### 4.4 UPDATE

```sql
-- ساده
UPDATE users SET email = 'newemail@example.com' WHERE id = 1;

-- چند ستون
UPDATE users 
SET 
    email = 'updated@example.com',
    is_active = FALSE
WHERE username = 'john_doe';

-- با RETURNING
UPDATE users SET is_active = FALSE 
WHERE id = 5
RETURNING *;
```

### 4.5 DELETE

```sql
-- حذف با شرط
DELETE FROM users WHERE id = 10;

-- حذف همه (خطرناک!)
DELETE FROM users;

-- با RETURNING
DELETE FROM users WHERE is_active = FALSE
RETURNING username;
```

---

## فصل 5: JOIN ها

### 5.1 INNER JOIN

```sql
SELECT 
    users.username,
    posts.title,
    posts.created_at
FROM users
INNER JOIN posts ON users.id = posts.user_id;

-- یا با alias
SELECT 
    u.username,
    p.title,
    p.created_at
FROM users u
INNER JOIN posts p ON u.id = p.user_id;
```

### 5.2 LEFT JOIN

```sql
-- همه کاربران + پست‌هاشون (حتی اگر پستی نداشته باشند)
SELECT 
    u.username,
    p.title
FROM users u
LEFT JOIN posts p ON u.id = p.user_id;
```

### 5.3 RIGHT JOIN

```sql
-- همه پست‌ها + کاربرشون
SELECT 
    u.username,
    p.title
FROM users u
RIGHT JOIN posts p ON u.id = p.user_id;
```

### 5.4 FULL OUTER JOIN

```sql
-- همه کاربران و همه پست‌ها
SELECT 
    u.username,
    p.title
FROM users u
FULL OUTER JOIN posts p ON u.id = p.user_id;
```

---

## فصل 6: توابع تجمیعی (Aggregate Functions)

### 6.1 توابع پایه

```sql
-- تعداد
SELECT COUNT(*) FROM users;
SELECT COUNT(DISTINCT email) FROM users;

-- مجموع
SELECT SUM(price) FROM orders;

-- میانگین
SELECT AVG(price) FROM orders;

-- حداقل و حداکثر
SELECT MIN(price), MAX(price) FROM orders;
```

### 6.2 GROUP BY

```sql
-- تعداد پست‌های هر کاربر
SELECT 
    user_id,
    COUNT(*) as post_count
FROM posts
GROUP BY user_id;

-- با JOIN
SELECT 
    u.username,
    COUNT(p.id) as post_count
FROM users u
LEFT JOIN posts p ON u.id = p.user_id
GROUP BY u.id, u.username;
```

### 6.3 HAVING

```sql
-- کاربرانی که بیشتر از 5 پست دارند
SELECT 
    user_id,
    COUNT(*) as post_count
FROM posts
GROUP BY user_id
HAVING COUNT(*) > 5;
```

---

## فصل 7: Subquery (زیرکوئری)

### 7.1 در WHERE

```sql
-- کاربرانی که پست دارند
SELECT * FROM users
WHERE id IN (SELECT DISTINCT user_id FROM posts);

-- کاربرانی که پست ندارند
SELECT * FROM users
WHERE id NOT IN (SELECT user_id FROM posts WHERE user_id IS NOT NULL);
```

### 7.2 در SELECT

```sql
SELECT 
    u.username,
    (SELECT COUNT(*) FROM posts WHERE user_id = u.id) as post_count
FROM users u;
```

### 7.3 در FROM

```sql
SELECT username, post_count
FROM (
    SELECT 
        u.username,
        COUNT(p.id) as post_count
    FROM users u
    LEFT JOIN posts p ON u.id = p.user_id
    GROUP BY u.username
) as user_stats
WHERE post_count > 3;
```

---

## فصل 8: Index (فهرست)

### 8.1 ایجاد Index

```sql
-- Index ساده
CREATE INDEX idx_users_email ON users(email);

-- Index یکتا
CREATE UNIQUE INDEX idx_users_username ON users(username);

-- Index چند ستونی
CREATE INDEX idx_posts_user_date ON posts(user_id, created_at);

-- MS SQL مشابه است فقط سینتکس کمی متفاوت
```

### 8.2 انواع Index در PostgreSQL

```sql
-- B-tree (پیش‌فرض)
CREATE INDEX idx_users_id ON users USING BTREE (id);

-- Hash (برای مقایسه مساوی)
CREATE INDEX idx_users_email_hash ON users USING HASH (email);

-- GIN (برای JSONB و آرایه‌ها)
CREATE INDEX idx_data_json ON my_table USING GIN (json_column);
```

### 8.3 حذف Index

```sql
DROP INDEX idx_users_email;
```

---

## فصل 9: View (نما)

### 9.1 ایجاد View

```sql
CREATE VIEW active_users AS
SELECT id, username, email
FROM users
WHERE is_active = TRUE;

-- استفاده از View
SELECT * FROM active_users;
```

### 9.2 Materialized View

```sql
-- View که نتایج رو ذخیره می‌کنه (سریع‌تر)
CREATE MATERIALIZED VIEW user_post_stats AS
SELECT 
    u.username,
    COUNT(p.id) as post_count
FROM users u
LEFT JOIN posts p ON u.id = p.user_id
GROUP BY u.username;

-- به‌روزرسانی Materialized View
REFRESH MATERIALIZED VIEW user_post_stats;

-- MS SQL معادل:
-- Indexed View یا Table با Trigger
```

---

## فصل 10: Functions (توابع)

### 10.1 توابع رشته‌ای

```sql
-- بزرگ/کوچک کردن
SELECT UPPER('hello'), LOWER('WORLD');

-- طول رشته
SELECT LENGTH('Hello World'); -- 11

-- قسمتی از رشته
SELECT SUBSTRING('Hello World' FROM 1 FOR 5); -- 'Hello'

-- جایگزینی
SELECT REPLACE('Hello World', 'World', 'PostgreSQL');

-- ترکیب رشته‌ها
SELECT CONCAT('Hello', ' ', 'World');
SELECT 'Hello' || ' ' || 'World'; -- عملگر || در PostgreSQL
```

### 10.2 توابع عددی

```sql
-- گرد کردن
SELECT ROUND(3.14159, 2); -- 3.14
SELECT CEIL(3.14), FLOOR(3.99); -- 4, 3

-- قدر مطلق
SELECT ABS(-15); -- 15

-- توان
SELECT POWER(2, 3); -- 8
```

### 10.3 توابع تاریخ و زمان

```sql
-- تاریخ و زمان فعلی
SELECT NOW(); -- TIMESTAMP با timezone
SELECT CURRENT_DATE; -- فقط تاریخ
SELECT CURRENT_TIME; -- فقط زمان

-- استخراج قسمت‌های تاریخ
SELECT EXTRACT(YEAR FROM NOW());
SELECT EXTRACT(MONTH FROM NOW());
SELECT EXTRACT(DAY FROM NOW());

-- MS SQL معادل:
-- SELECT GETDATE() به جای NOW()
-- SELECT YEAR(GETDATE()) به جای EXTRACT

-- محاسبات با تاریخ
SELECT NOW() + INTERVAL '7 days';
SELECT NOW() - INTERVAL '1 month';

-- MS SQL معادل:
-- SELECT DATEADD(DAY, 7, GETDATE())

-- تفاوت تاریخ‌ها
SELECT AGE(TIMESTAMP '2024-01-01', TIMESTAMP '2023-01-01');
```

### 10.4 تبدیل نوع داده

```sql
-- CAST
SELECT CAST('123' AS INTEGER);
SELECT CAST(123 AS VARCHAR);

-- سینتکس کوتاه PostgreSQL
SELECT '123'::INTEGER;
SELECT 123::VARCHAR;

-- MS SQL:
-- SELECT CAST('123' AS INT)
-- SELECT CONVERT(INT, '123')
```

---

## فصل 11: Stored Procedures & Functions

### 11.1 تابع ساده (Function)

```sql
-- PostgreSQL
CREATE OR REPLACE FUNCTION get_user_count()
RETURNS INTEGER AS $$
BEGIN
    RETURN (SELECT COUNT(*) FROM users);
END;
$$ LANGUAGE plpgsql;

-- استفاده
SELECT get_user_count();

-- MS SQL معادل:
-- CREATE FUNCTION get_user_count()
-- RETURNS INT AS
-- BEGIN
--     RETURN (SELECT COUNT(*) FROM users);
-- END;
```

### 11.2 تابع با پارامتر

```sql
CREATE OR REPLACE FUNCTION get_user_posts(p_user_id INTEGER)
RETURNS TABLE(post_id INTEGER, post_title VARCHAR) AS $$
BEGIN
    RETURN QUERY
    SELECT id, title FROM posts WHERE user_id = p_user_id;
END;
$$ LANGUAGE plpgsql;

-- استفاده
SELECT * FROM get_user_posts(5);
```

### 11.3 Procedure

```sql
-- PostgreSQL (از نسخه 11 به بعد)
CREATE OR REPLACE PROCEDURE add_user(
    p_username VARCHAR,
    p_email VARCHAR
)
LANGUAGE plpgsql
AS $$
BEGIN
    INSERT INTO users (username, email) 
    VALUES (p_username, p_email);
    COMMIT;
END;
$$;

-- استفاده
CALL add_user('newuser', 'new@example.com');

-- MS SQL:
-- CREATE PROCEDURE add_user
--     @username VARCHAR(50),
--     @email VARCHAR(100)
-- AS
-- BEGIN
--     INSERT INTO users...
-- END;
-- EXEC add_user 'newuser', 'new@example.com'
```

### 11.4 تابع با شرط‌ها

```sql
CREATE OR REPLACE FUNCTION check_user_status(p_user_id INTEGER)
RETURNS VARCHAR AS $$
DECLARE
    v_post_count INTEGER;
BEGIN
    SELECT COUNT(*) INTO v_post_count 
    FROM posts 
    WHERE user_id = p_user_id;
    
    IF v_post_count > 10 THEN
        RETURN 'Active';
    ELSIF v_post_count > 0 THEN
        RETURN 'Normal';
    ELSE
        RETURN 'Inactive';
    END IF;
END;
$$ LANGUAGE plpgsql;
```

---

## فصل 12: Trigger (ماشه)

### 12.1 ایجاد Trigger

```sql
-- ابتدا تابع Trigger را می‌سازیم
CREATE OR REPLACE FUNCTION update_modified_timestamp()
RETURNS TRIGGER AS $$
BEGIN
    NEW.updated_at = NOW();
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

-- سپس Trigger را به جدول متصل می‌کنیم
CREATE TRIGGER trg_users_update
    BEFORE UPDATE ON users
    FOR EACH ROW
    EXECUTE FUNCTION update_modified_timestamp();

-- MS SQL تفاوت دارد:
-- CREATE TRIGGER trg_users_update
-- ON users
-- AFTER UPDATE
-- AS
-- BEGIN
--     UPDATE users SET updated_at = GETDATE()
--     WHERE id IN (SELECT id FROM inserted);
-- END;
```

### 12.2 انواع Trigger

```sql
-- BEFORE INSERT
CREATE TRIGGER trg_before_insert
    BEFORE INSERT ON posts
    FOR EACH ROW
    EXECUTE FUNCTION my_function();

-- AFTER INSERT
CREATE TRIGGER trg_after_insert
    AFTER INSERT ON posts
    FOR EACH ROW
    EXECUTE FUNCTION my_function();

-- INSTEAD OF (فقط برای View)
CREATE TRIGGER trg_instead_of
    INSTEAD OF INSERT ON my_view
    FOR EACH ROW
    EXECUTE FUNCTION my_function();
```

---

## فصل 13: Transaction (تراکنش)

### 13.1 مفهوم Transaction

```sql
-- شروع تراکنش
BEGIN;

-- عملیات‌ها
INSERT INTO users (username, email) VALUES ('test', 'test@test.com');
UPDATE posts SET title = 'New Title' WHERE id = 5;

-- تایید تراکنش
COMMIT;

-- یا لغو تراکنش
-- ROLLBACK;
```

### 13.2 Savepoint

```sql
BEGIN;

INSERT INTO users (username, email) VALUES ('user1', 'user1@test.com');

SAVEPOINT sp1;

INSERT INTO users (username, email) VALUES ('user2', 'user2@test.com');

-- اگر مشکلی بود، فقط تا savepoint برگرد
ROLLBACK TO sp1;

COMMIT;
```

### 13.3 سطوح Isolation

```sql
-- READ COMMITTED (پیش‌فرض)
BEGIN TRANSACTION ISOLATION LEVEL READ COMMITTED;

-- REPEATABLE READ
BEGIN TRANSACTION ISOLATION LEVEL REPEATABLE READ;

-- SERIALIZABLE
BEGIN TRANSACTION ISOLATION LEVEL SERIALIZABLE;

-- MS SQL مشابه است
```

---

## فصل 14: ویژگی‌های خاص PostgreSQL

### 14.1 ARRAY (آرایه)

```sql
-- ایجاد جدول با ستون آرایه
CREATE TABLE products (
    id SERIAL PRIMARY KEY,
    name VARCHAR(100),
    tags TEXT[]
);

-- درج داده
INSERT INTO products (name, tags) 
VALUES ('Laptop', ARRAY['electronics', 'computers', 'tech']);

-- یا
INSERT INTO products (name, tags) 
VALUES ('Phone', '{"electronics", "mobile", "tech"}');

-- جستجو در آرایه
SELECT * FROM products WHERE 'tech' = ANY(tags);
SELECT * FROM products WHERE tags @> ARRAY['electronics'];

-- MS SQL آرایه ندارد، باید از جدول جداگانه استفاده کنی
```

### 14.2 JSON/JSONB

```sql
-- ایجاد جدول
CREATE TABLE users_data (
    id SERIAL PRIMARY KEY,
    username VARCHAR(50),
    metadata JSONB
);

-- درج داده
INSERT INTO users_data (username, metadata)
VALUES ('john', '{"age": 30, "city": "Tehran", "hobbies": ["reading", "coding"]}');

-- استخراج از JSON
SELECT 
    username,
    metadata->>'age' as age,
    metadata->>'city' as city
FROM users_data;

-- جستجو در JSON
SELECT * FROM users_data WHERE metadata->>'city' = 'Tehran';
SELECT * FROM users_data WHERE metadata->'age' > '25';

-- بررسی وجود کلید
SELECT * FROM users_data WHERE metadata ? 'age';

-- MS SQL:
-- JSON_VALUE(metadata, '$.age')
-- ولی PostgreSQL قدرتمندتر است
```

### 14.3 ENUM (شمارشی)

```sql
-- ایجاد نوع ENUM
CREATE TYPE user_role AS ENUM ('admin', 'moderator', 'user');

-- استفاده در جدول
CREATE TABLE users (
    id SERIAL PRIMARY KEY,
    username VARCHAR(50),
    role user_role DEFAULT 'user'
);

INSERT INTO users (username, role) VALUES ('admin_user', 'admin');

-- MS SQL معادل ندارد، باید از CHECK constraint استفاده کنی
```

### 14.4 Full Text Search

```sql
-- ایجاد ستون tsvector
ALTER TABLE posts ADD COLUMN search_vector tsvector;

-- به‌روزرسانی
UPDATE posts SET search_vector = 
    to_tsvector('english', title || ' ' || content);

-- جستجو
SELECT * FROM posts 
WHERE search_vector @@ to_tsquery('english', 'PostgreSQL & tutorial');

-- Index برای سرعت
CREATE INDEX idx_posts_search ON posts USING GIN(search_vector);
```

---

## فصل 15: بهینه‌سازی و Performance

### 15.1 EXPLAIN (تحلیل کوئری)

```sql
-- نمایش برنامه اجرای کوئری
EXPLAIN SELECT * FROM users WHERE email = 'test@test.com';

-- با جزئیات بیشتر
EXPLAIN ANALYZE SELECT * FROM users WHERE email = 'test@test.com';

-- MS SQL معادل:
-- SET SHOWPLAN_ALL ON
-- یا استفاده از Execution Plan در SSMS
```

### 15.2 VACUUM

```sql
-- پاکسازی فضای اضافی
VACUUM users;

-- با تحلیل آماری
VACUUM ANALYZE users;

-- بازسازی کامل جدول
VACUUM FULL users;

-- MS SQL معادل:
-- DBCC SHRINKFILE برای فضا
-- UPDATE STATISTICS برای آمار
```

### 15.3 نکات بهینه‌سازی

```sql
-- 1. استفاده از Index مناسب
CREATE INDEX idx_users_email ON users(email);

-- 2. محدود کردن ستون‌های SELECT
SELECT id, username FROM users; -- به جای SELECT *

-- 3. استفاده از EXISTS به جای COUNT
SELECT EXISTS(SELECT 1 FROM users WHERE id = 5); -- سریع‌تر

-- 4. استفاده از LIMIT
SELECT * FROM posts ORDER BY created_at DESC LIMIT 100;

-- 5. JOIN به جای Subquery (در اکثر موارد)
-- بهتر:
SELECT u.*, p.title FROM users u
JOIN posts p ON u.id = p.user_id;

-- به جای:
SELECT *, (SELECT title FROM posts WHERE user_id = u.id LIMIT 1)
FROM users u;
```

---

## فصل 16: Constraint (محدودیت‌ها)

### 16.1 PRIMARY KEY

```sql
CREATE TABLE users (
    id SERIAL PRIMARY KEY,
    username VARCHAR(50)
);

-- یا
ALTER TABLE users ADD PRIMARY KEY (id);
```

### 16.2 FOREIGN KEY

```sql
CREATE TABLE posts (
    id SERIAL PRIMARY KEY,
    user_id INTEGER REFERENCES users(id)
);

-- با گزینه‌های بیشتر
ALTER TABLE posts
ADD CONSTRAINT fk_posts_user
FOREIGN KEY (user_id) REFERENCES users(id)
ON DELETE CASCADE
ON UPDATE CASCADE;
```

### 16.3 UNIQUE

```sql
ALTER TABLE users ADD UNIQUE (email);

-- یا
ALTER TABLE users 
ADD CONSTRAINT uq_users_email UNIQUE (email);
```

### 16.4 CHECK

```sql
ALTER TABLE users 
ADD CONSTRAINT chk_age CHECK (age >= 18);

-- چند شرط
ALTER TABLE products
ADD CONSTRAINT chk_price CHECK (price > 0 AND price < 1000000);
```

### 16.5 NOT NULL

```sql
ALTER TABLE users ALTER COLUMN email SET NOT NULL;

-- برداشتن NOT NULL
ALTER TABLE users ALTER COLUMN phone DROP NOT NULL;
```

### 16.6 DEFAULT

```sql
ALTER TABLE users ALTER COLUMN is_active SET DEFAULT TRUE;
ALTER TABLE users ALTER COLUMN created_at SET DEFAULT NOW();
```

---

## فصل 17: Window Functions (توابع پنجره‌ای)

### 17.1 ROW_NUMBER

```sql
SELECT 
    username,
    email,
    ROW_NUMBER() OVER (ORDER BY created_at) as row_num
FROM users;

-- MS SQL مشابه است
```

### 17.2 RANK و DENSE_RANK

```sql
SELECT 
    username,
    score,
    RANK() OVER (ORDER BY score DESC) as rank,
    DENSE_RANK() OVER (ORDER BY score DESC) as dense_rank
FROM user_scores;
```

### 17.3 PARTITION BY

```sql
SELECT 
    user_id,
    title,
    created_at,
    ROW_NUMBER() OVER (PARTITION BY user_id ORDER BY created_at DESC) as post_rank
FROM posts;

-- پست آخر هر کاربر
SELECT * FROM (
    SELECT 
        user_id,
        title,
        ROW_NUMBER() OVER (PARTITION BY user_id ORDER BY created_at DESC) as rn
    FROM posts
) sub
WHERE rn = 1;
```

### 17.4 LAG و LEAD

```sql
-- مقایسه با رکورد قبلی
SELECT 
    date,
    sales,
    LAG(sales) OVER (ORDER BY date) as previous_sales,
    sales - LAG(sales) OVER (ORDER BY date) as difference
FROM daily_sales;

-- مقایسه با رکورد بعدی
SELECT 
    date,
    sales,
    LEAD(sales) OVER (ORDER BY date) as next_sales
FROM daily_sales;
```

---

## فصل 18: CTE (Common Table Expression)

### 18.1 CTE ساده

```sql
WITH active_users AS (
    SELECT * FROM users WHERE is_active = TRUE
)
SELECT * FROM active_users WHERE created_at > '2024-01-01';

-- MS SQL هم از CTE پشتیبانی می‌کند
```

### 18.2 چند CTE

```sql
WITH 
active_users AS (
    SELECT * FROM users WHERE is_active = TRUE
),
recent_posts AS (
    SELECT * FROM posts WHERE created_at > NOW() - INTERVAL '30 days'
)
SELECT 
    u.username,
    COUNT(p.id) as post_count
FROM active_users u
LEFT JOIN recent_posts p ON u.id = p.user_id
GROUP BY u.username;
```

### 18.3 Recursive CTE

```sql
-- ساختار سلسله مراتبی
CREATE TABLE employees (
    id SERIAL PRIMARY KEY,
    name VARCHAR(100),
    manager_id INTEGER REFERENCES employees(id)
);

-- پیدا کردن همه زیردستان
WITH RECURSIVE employee_hierarchy AS (
    -- شروع از مدیر ارشد
    SELECT id, name, manager_id, 1 as level
    FROM employees
    WHERE manager_id IS NULL
    
    UNION ALL
    
    -- پیدا کردن زیردستان
    SELECT e.id, e.name, e.manager_id, eh.level + 1
    FROM employees e
    JOIN employee_hierarchy eh ON e.manager_id = eh.id
)
SELECT * FROM employee_hierarchy ORDER BY level, name;

-- MS SQL هم از Recursive CTE پشتیبانی می‌کند
```

---

## فصل 19: CASE (شرط چندگانه)

### 19.1 CASE ساده

```sql
SELECT 
    username,
    is_active,
    CASE 
        WHEN is_active = TRUE THEN 'فعال'
        WHEN is_active = FALSE THEN 'غیرفعال'
        ELSE 'نامشخص'
    END as status
FROM users;

-- MS SQL مشابه است
```

### 19.2 CASE با محاسبات

```sql
SELECT 
    username,
    (SELECT COUNT(*) FROM posts WHERE user_id = u.id) as post_count,
    CASE 
        WHEN (SELECT COUNT(*) FROM posts WHERE user_id = u.id) > 50 THEN 'پرکار'
        WHEN (SELECT COUNT(*) FROM posts WHERE user_id = u.id) > 10 THEN 'متوسط'
        ELSE 'کم‌کار'
    END as activity_level
FROM users u;
```

### 19.3 CASE در ORDER BY

```sql
SELECT * FROM users
ORDER BY 
    CASE 
        WHEN is_active = TRUE THEN 1
        ELSE 2
    END,
    username;
```

---

## فصل 20: UNION, INTERSECT, EXCEPT

### 20.1 UNION (ترکیب)

```sql
-- UNION (بدون تکراری)
SELECT email FROM users
UNION
SELECT email FROM customers;

-- UNION ALL (با تکراری)
SELECT email FROM users
UNION ALL
SELECT email FROM customers;

-- MS SQL مشابه است
```

### 20.2 INTERSECT (اشتراک)

```sql
-- ایمیل‌هایی که هم در users و هم در customers هستند
SELECT email FROM users
INTERSECT
SELECT email FROM customers;
```

### 20.3 EXCEPT (تفاضل)

```sql
-- ایمیل‌هایی که در users هستند ولی در customers نیستند
SELECT email FROM users
EXCEPT
SELECT email FROM customers;

-- MS SQL از EXCEPT پشتیبانی می‌کند
```

---

## فصل 21: Sequence (دنباله)

### 21.1 ایجاد Sequence

```sql
-- ایجاد
CREATE SEQUENCE user_id_seq
    START WITH 1
    INCREMENT BY 1
    MINVALUE 1
    MAXVALUE 999999
    CACHE 1;

-- استفاده
INSERT INTO users (id, username) 
VALUES (nextval('user_id_seq'), 'newuser');

-- دریافت مقدار فعلی
SELECT currval('user_id_seq');

-- MS SQL:
-- CREATE SEQUENCE user_id_seq START WITH 1 INCREMENT BY 1;
-- NEXT VALUE FOR user_id_seq
```

### 21.2 تنظیم مجدد Sequence

```sql
-- تنظیم مجدد به 1
ALTER SEQUENCE user_id_seq RESTART WITH 1;

-- تنظیم به آخرین مقدار جدول
SELECT setval('user_id_seq', (SELECT MAX(id) FROM users));
```

---

## فصل 22: Schema Management

### 22.1 ایجاد و استفاده از Schema

```sql
-- ایجاد Schema
CREATE SCHEMA sales;
CREATE SCHEMA hr;

-- ایجاد جدول در Schema
CREATE TABLE sales.orders (
    id SERIAL PRIMARY KEY,
    amount NUMERIC(10,2)
);

CREATE TABLE hr.employees (
    id SERIAL PRIMARY KEY,
    name VARCHAR(100)
);

-- تنظیم Schema پیش‌فرض
SET search_path TO sales, public;

-- بعد از این می‌توانید بدون نام Schema استفاده کنید
SELECT * FROM orders; -- از sales.orders می‌خواند
```

### 22.2 لیست Schema ها

```sql
-- نمایش همه Schema ها
SELECT schema_name FROM information_schema.schemata;

-- نمایش جداول یک Schema
SELECT table_name 
FROM information_schema.tables 
WHERE table_schema = 'sales';
```

---

## فصل 23: User Management و Security

### 23.1 ایجاد کاربر (Role)

```sql
-- ایجاد کاربر با رمز عبور
CREATE ROLE john_user WITH LOGIN PASSWORD 'secure_password';

-- ایجاد کاربر با دسترسی‌های خاص
CREATE ROLE readonly_user WITH LOGIN PASSWORD 'pass123'
    VALID UNTIL '2025-12-31';

-- MS SQL:
-- CREATE LOGIN john_user WITH PASSWORD = 'secure_password';
-- CREATE USER john_user FOR LOGIN john_user;
```

### 23.2 دادن دسترسی (GRANT)

```sql
-- دسترسی به دیتابیس
GRANT CONNECT ON DATABASE my_database TO john_user;

-- دسترسی به Schema
GRANT USAGE ON SCHEMA public TO john_user;

-- دسترسی SELECT به جدول
GRANT SELECT ON users TO john_user;

-- دسترسی کامل به جدول
GRANT ALL PRIVILEGES ON users TO john_user;

-- دسترسی به همه جداول در Schema
GRANT SELECT ON ALL TABLES IN SCHEMA public TO readonly_user;

-- MS SQL:
-- GRANT SELECT ON users TO john_user;
```

### 23.3 گرفتن دسترسی (REVOKE)

```sql
-- گرفتن دسترسی
REVOKE SELECT ON users FROM john_user;

-- گرفتن همه دسترسی‌ها
REVOKE ALL PRIVILEGES ON users FROM john_user;
```

### 23.4 Role های گروهی

```sql
-- ایجاد Role گروهی
CREATE ROLE developers;
GRANT SELECT, INSERT, UPDATE ON ALL TABLES IN SCHEMA public TO developers;

-- اضافه کردن کاربر به گروه
GRANT developers TO john_user;
GRANT developers TO alice_user;
```

---

## فصل 24: Backup و Restore

### 24.1 Backup با pg_dump

```bash
# Backup یک دیتابیس
pg_dump -U username -d database_name > backup.sql

# Backup با فرمت custom (فشرده‌تر و سریع‌تر)
pg_dump -U username -d database_name -F c -f backup.dump

# Backup فقط Schema (بدون داده)
pg_dump -U username -d database_name --schema-only > schema.sql

# Backup فقط داده (بدون Schema)
pg_dump -U username -d database_name --data-only > data.sql

# Backup یک جدول خاص
pg_dump -U username -d database_name -t users > users_backup.sql

# MS SQL معادل:
# BACKUP DATABASE database_name TO DISK = 'C:\backup.bak'
```

### 24.2 Restore

```bash
# Restore از فایل SQL
psql -U username -d database_name < backup.sql

# Restore از فایل custom
pg_restore -U username -d database_name backup.dump

# Restore با پاک کردن داده‌های قبلی
pg_restore -U username -d database_name --clean backup.dump

# MS SQL معادل:
# RESTORE DATABASE database_name FROM DISK = 'C:\backup.bak'
```

### 24.3 Backup همه دیتابیس‌ها

```bash
# Backup تمام دیتابیس‌ها
pg_dumpall -U postgres > all_databases.sql

# Restore
psql -U postgres < all_databases.sql
```

---

## فصل 25: کار با فایل‌های CSV

### 25.1 Import از CSV

```sql
-- Import با COPY
COPY users(username, email, created_at)
FROM '/path/to/users.csv'
DELIMITER ','
CSV HEADER;

-- Import با \copy در psql (بدون نیاز به دسترسی superuser)
\copy users(username, email) FROM 'users.csv' DELIMITER ',' CSV HEADER;

-- MS SQL معادل:
-- BULK INSERT users FROM 'users.csv'
-- WITH (FIELDTERMINATOR = ',', ROWTERMINATOR = '\n', FIRSTROW = 2);
```

### 25.2 Export به CSV

```sql
-- Export با COPY
COPY users TO '/path/to/users_export.csv' DELIMITER ',' CSV HEADER;

-- Export نتیجه Query
COPY (SELECT username, email FROM users WHERE is_active = TRUE)
TO '/path/to/active_users.csv' DELIMITER ',' CSV HEADER;

-- با \copy در psql
\copy (SELECT * FROM users) TO 'users.csv' CSV HEADER;
```

---

## فصل 26: Extensions (افزونه‌ها)

### 26.1 مشاهده و نصب Extension

```sql
-- لیست Extensions نصب شده
SELECT * FROM pg_extension;

-- لیست Extensions موجود
SELECT * FROM pg_available_extensions;

-- نصب Extension
CREATE EXTENSION IF NOT EXISTS "uuid-ossp";
CREATE EXTENSION IF NOT EXISTS "pg_trgm"; -- برای جستجوی متن
CREATE EXTENSION IF NOT EXISTS "postgis"; -- برای GIS

-- MS SQL معادل ندارد، باید از CLR استفاده کنی
```

### 26.2 استفاده از uuid-ossp

```sql
-- ایجاد UUID
SELECT uuid_generate_v4();

-- استفاده در جدول
CREATE TABLE users (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    username VARCHAR(50)
);
```

### 26.3 استفاده از pg_trgm (جستجوی فازی)

```sql
-- فعال‌سازی
CREATE EXTENSION pg_trgm;

-- جستجوی شباهت
SELECT * FROM users 
WHERE username % 'john'; -- پیدا کردن نام‌های شبیه john

-- Index برای جستجوی سریع‌تر
CREATE INDEX idx_users_username_trgm ON users USING GIN (username gin_trgm_ops);
```

---

## فصل 27: Information Schema

### 27.1 اطلاعات جداول

```sql
-- لیست همه جداول
SELECT table_schema, table_name 
FROM information_schema.tables 
WHERE table_schema NOT IN ('pg_catalog', 'information_schema')
ORDER BY table_schema, table_name;

-- لیست ستون‌های یک جدول
SELECT column_name, data_type, character_maximum_length
FROM information_schema.columns
WHERE table_name = 'users';

-- MS SQL:
-- SELECT * FROM INFORMATION_SCHEMA.TABLES
-- SELECT * FROM INFORMATION_SCHEMA.COLUMNS WHERE TABLE_NAME = 'users'
```

### 27.2 اطلاعات Constraints

```sql
-- لیست Foreign Keys
SELECT
    tc.table_name,
    kcu.column_name,
    ccu.table_name AS foreign_table_name,
    ccu.column_name AS foreign_column_name
FROM information_schema.table_constraints AS tc
JOIN information_schema.key_column_usage AS kcu
    ON tc.constraint_name = kcu.constraint_name
JOIN information_schema.constraint_column_usage AS ccu
    ON ccu.constraint_name = tc.constraint_name
WHERE tc.constraint_type = 'FOREIGN KEY';
```

### 27.3 اطلاعات Index ها

```sql
-- لیست Index ها
SELECT
    tablename,
    indexname,
    indexdef
FROM pg_indexes
WHERE schemaname = 'public'
ORDER BY tablename, indexname;
```

---

## فصل 28: نکات پیشرفته

### 28.1 UPSERT (INSERT ... ON CONFLICT)

```sql
-- اگر username تکراری بود، email را به‌روز کن
INSERT INTO users (username, email) 
VALUES ('john', 'john@example.com')
ON CONFLICT (username) 
DO UPDATE SET email = EXCLUDED.email;

-- یا نادیده بگیر
INSERT INTO users (username, email) 
VALUES ('john', 'john@example.com')
ON CONFLICT (username) 
DO NOTHING;

-- MS SQL معادل:
-- MERGE یا IF EXISTS ... UPDATE ELSE INSERT
```

### 28.2 LATERAL JOIN

```sql
-- برای هر کاربر، 3 پست آخرش را بیاور
SELECT 
    u.username,
    p.title,
    p.created_at
FROM users u
LEFT JOIN LATERAL (
    SELECT * FROM posts 
    WHERE user_id = u.id 
    ORDER BY created_at DESC 
    LIMIT 3
) p ON TRUE;

-- MS SQL معادل:
-- CROSS APPLY یا OUTER APPLY
```

### 28.3 DISTINCT ON

```sql
-- اولین پست هر کاربر
SELECT DISTINCT ON (user_id) 
    user_id,
    title,
    created_at
FROM posts
ORDER BY user_id, created_at ASC;

-- MS SQL معادل ندارد، باید از ROW_NUMBER استفاده کنی
```

### 28.4 RETURNING در DELETE و UPDATE

```sql
-- حذف و دریافت اطلاعات رکوردهای حذف شده
DELETE FROM users 
WHERE is_active = FALSE
RETURNING id, username, email;

-- به‌روزرسانی و دریافت نتیجه
UPDATE users 
SET is_active = TRUE 
WHERE id IN (1,2,3)
RETURNING *;
```

### 28.5 WITH CHECK OPTION در View

```sql
-- ایجاد View با محدودیت
CREATE VIEW active_users AS
SELECT * FROM users WHERE is_active = TRUE
WITH CHECK OPTION;

-- حالا نمی‌توانید از طریق View رکورد غیرفعال اضافه کنید
-- این دستور خطا می‌دهد:
-- INSERT INTO active_users (username, email, is_active) 
-- VALUES ('test', 'test@test.com', FALSE);
```

---

## فصل 29: تفاوت‌های مهم PostgreSQL و MS SQL

### 29.1 سینتکس

| عملیات | PostgreSQL | MS SQL |
|--------|-----------|--------|
| محدود کردن تعداد | `LIMIT 10` | `TOP 10` |
| رشته‌های متنی | `'text'` یا `$text$` | `'text'` یا `N'text'` |
| ترکیب رشته | `'a' \|\| 'b'` | `'a' + 'b'` |
| تاریخ فعلی | `NOW()` یا `CURRENT_TIMESTAMP` | `GETDATE()` |
| شماره خودکار | `SERIAL` | `IDENTITY` |
| بولین | `TRUE/FALSE` | `1/0` یا `BIT` |
| IF شرطی | در Functions/Procedures | `IF ... BEGIN ... END` |

### 29.2 ویژگی‌های منحصر به PostgreSQL

- **Array ها**: PostgreSQL از آرایه پشتیبانی می‌کند
- **JSONB**: نوع داده JSON بهینه‌شده
- **Inheritance**: جداول می‌توانند از هم ارث‌بری کنند
- **Custom Types**: می‌توانید نوع داده خودتان را بسازید
- **Procedural Languages**: PL/Python, PL/Perl, PL/R
- **Full Text Search**: جستجوی متن کامل داخلی
- **PostGIS**: پشتیبانی از داده‌های مکانی
- **Listen/Notify**: سیستم پیام‌رسانی

### 29.3 ویژگی‌های منحصر به MS SQL

- **T-SQL**: زبان قدرتمندتر برای Stored Procedures
- **SSMS**: رابط کاربری قدرتمند
- **Integration Services**: ETL داخلی
- **Reporting Services**: گزارش‌گیری داخلی
- **ادغام با .NET**: پشتیبانی از CLR

---

## فصل 30: تمرین‌های عملی

### تمرین 1: سیستم وبلاگ

```sql
-- ایجاد جداول
CREATE TABLE authors (
    id SERIAL PRIMARY KEY,
    name VARCHAR(100) NOT NULL,
    email VARCHAR(100) UNIQUE NOT NULL,
    created_at TIMESTAMP DEFAULT NOW()
);

CREATE TABLE categories (
    id SERIAL PRIMARY KEY,
    name VARCHAR(50) NOT NULL UNIQUE,
    description TEXT
);

CREATE TABLE articles (
    id SERIAL PRIMARY KEY,
    author_id INTEGER REFERENCES authors(id) ON DELETE CASCADE,
    category_id INTEGER REFERENCES categories(id) ON DELETE SET NULL,
    title VARCHAR(200) NOT NULL,
    content TEXT NOT NULL,
    views INTEGER DEFAULT 0,
    published_at TIMESTAMP,
    created_at TIMESTAMP DEFAULT NOW()
);

CREATE TABLE comments (
    id SERIAL PRIMARY KEY,
    article_id INTEGER REFERENCES articles(id) ON DELETE CASCADE,
    author_name VARCHAR(100) NOT NULL,
    content TEXT NOT NULL,
    created_at TIMESTAMP DEFAULT NOW()
);

-- درج داده نمونه
INSERT INTO authors (name, email) VALUES 
    ('علی محمدی', 'ali@example.com'),
    ('سارا احمدی', 'sara@example.com');

INSERT INTO categories (name, description) VALUES 
    ('فناوری', 'مقالات مرتبط با فناوری'),
    ('برنامه‌نویسی', 'مقالات برنامه‌نویسی');

INSERT INTO articles (author_id, category_id, title, content, views, published_at) VALUES 
    (1, 1, 'آموزش PostgreSQL', 'محتوای مقاله...', 100, NOW()),
    (2, 2, 'آموزش Python', 'محتوای مقاله...', 50, NOW());

-- کوئری‌های کاربردی

-- محبوب‌ترین مقالات
SELECT 
    a.title,
    au.name as author,
    c.name as category,
    a.views
FROM articles a
JOIN authors au ON a.author_id = au.id
LEFT JOIN categories c ON a.category_id = c.id
ORDER BY a.views DESC
LIMIT 10;

-- نویسندگان پرکار
SELECT 
    au.name,
    COUNT(a.id) as article_count,
    SUM(a.views) as total_views
FROM authors au
LEFT JOIN articles a ON au.id = a.author_id
GROUP BY au.id, au.name
ORDER BY article_count DESC;

-- مقالات با تعداد کامنت
SELECT 
    a.title,
    au.name as author,
    COUNT(com.id) as comment_count
FROM articles a
JOIN authors au ON a.author_id = au.id
LEFT JOIN comments com ON a.id = com.article_id
GROUP BY a.id, a.title, au.name
ORDER BY comment_count DESC;
```

### تمرین 2: سیستم فروشگاه

```sql
-- جداول
CREATE TABLE customers (
    id SERIAL PRIMARY KEY,
    name VARCHAR(100) NOT NULL,
    email VARCHAR(100) UNIQUE,
    phone VARCHAR(15),
    created_at TIMESTAMP DEFAULT NOW()
);

CREATE TABLE products (
    id SERIAL PRIMARY KEY,
    name VARCHAR(100) NOT NULL,
    description TEXT,
    price NUMERIC(10,2) NOT NULL CHECK (price > 0),
    stock INTEGER DEFAULT 0 CHECK (stock >= 0),
    created_at TIMESTAMP DEFAULT NOW()
);

CREATE TABLE orders (
    id SERIAL PRIMARY KEY,
    customer_id INTEGER REFERENCES customers(id),
    order_date TIMESTAMP DEFAULT NOW(),
    total_amount NUMERIC(10,2),
    status VARCHAR(20) DEFAULT 'pending'
);

CREATE TABLE order_items (
    id SERIAL PRIMARY KEY,
    order_id INTEGER REFERENCES orders(id) ON DELETE CASCADE,
    product_id INTEGER REFERENCES products(id),
    quantity INTEGER NOT NULL CHECK (quantity > 0),
    price NUMERIC(10,2) NOT NULL
);

-- Function برای ثبت سفارش
CREATE OR REPLACE FUNCTION create_order(
    p_customer_id INTEGER,
    p_products INTEGER[],
    p_quantities INTEGER[]
)
RETURNS INTEGER AS $
DECLARE
    v_order_id INTEGER;
    v_total NUMERIC(10,2) := 0;
    v_price NUMERIC(10,2);
    i INTEGER;
BEGIN
    -- ایجاد سفارش
    INSERT INTO orders (customer_id, total_amount)
    VALUES (p_customer_id, 0)
    RETURNING id INTO v_order_id;
    
    -- اضافه کردن آیتم‌ها
    FOR i IN 1..array_length(p_products, 1) LOOP
        SELECT price INTO v_price FROM products WHERE id = p_products[i];
        
        INSERT INTO order_items (order_id, product_id, quantity, price)
        VALUES (v_order_id, p_products[i], p_quantities[i], v_price);
        
        v_total := v_total + (v_price * p_quantities[i]);
        
        -- کم کردن از موجودی
        UPDATE products 
        SET stock = stock - p_quantities[i]
        WHERE id = p_products[i];
    END LOOP;
    
    -- به‌روزرسانی مبلغ کل
    UPDATE orders SET total_amount = v_total WHERE id = v_order_id;
    
    RETURN v_order_id;
END;
$ LANGUAGE plpgsql;

-- استفاده
-- SELECT create_order(1, ARRAY[1,2], ARRAY[2,3]);
```

---

## منابع مفید

### مستندات رسمی
- [PostgreSQL Documentation](https://www.postgresql.org/docs/)
- [PostgreSQL Tutorial](https://www.postgresqltutorial.com/)

### ابزارها
- **pgAdmin**: رابط گرافیکی
- **DBeaver**: کلاینت چند پلتفرمی
- **DataGrip**: IDE حرفه‌ای (JetBrains)
- **psql**: Command Line

### نکات پایانی
1. همیشه از Transaction استفاده کنید
2. Index های مناسب بسازید
3. از EXPLAIN برای بهینه‌سازی استفاده کنید
4. Backup منظم بگیرید
5. Log های PostgreSQL را بررسی کنید
6. از Connection Pooling استفاده کنید (PgBouncer)
7. Configuration را بر اساس سخت‌افزار تنظیم کنید

---

**این جزوه پوشش جامعی از PostgreSQL می‌دهد. برای یادگیری بهتر:**
- تمرین‌های عملی انجام دهید
- پروژه واقعی بسازید
- مستندات رسمی را مطالعه کنید
- در جامعه PostgreSQL فعال باشید

موفق باشید! 🚀