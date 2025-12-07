### Login to PostgreSQL:
```
sudo -i -u postgres
psql
```
```
\c ecommerce_db
```

### table products 
```
CREATE TABLE products (
    product_id SERIAL PRIMARY KEY,
    product_name VARCHAR(100),
    price DECIMAL(10, 2),
    category VARCHAR(50)
);
```
```
\dt
```

### table orders 
```
CREATE TABLE orders (
    order_id SERIAL PRIMARY KEY,
    user_id INTEGER,
    order_date DATE DEFAULT CURRENT_DATE,
    total_amount DECIMAL(10, 2),
    status VARCHAR(20) DEFAULT 'pending'
);
```
```
\dt
```

### table order_items
```
CREATE TABLE order_items (
    order_item_id SERIAL PRIMARY KEY,
    order_id INTEGER REFERENCES orders(order_id),
    product_id INTEGER REFERENCES products(product_id),
    quantity INTEGER,
    unit_price DECIMAL(10, 2)
);
```
```
\dt
```
```
\! clear
```

### Insert sample data
```
INSERT INTO products (product_name, price, category) 
VALUES ('Laptop', 20000000, 'Electronics');

INSERT INTO products (product_name, price, category) VALUES
('Mouse', 350000, 'Electronics'),
('Book', 50000, 'Education'),
('Clothing', 300000, 'Apparel'),
('Snack', 20000, 'Food');
```
```
SELECT * FROM products;
```

### Create 1000 sample orders
```
INSERT INTO orders (user_id, order_date, total_amount, status)
SELECT 
    (random() * 1000000)::INTEGER + 1,
    CURRENT_DATE - (random() * 30)::INTEGER,
    (random() * 1000000) + 50000,
    'pending'
FROM generate_series(1, 1000);
```
```
SELECT COUNT(*) FROM orders;
```
```
\! clear
```

## Method 1: Without DP (simple but slow)

```
-- Function: Calculate total purchase per user (without DP)
CREATE OR REPLACE FUNCTION get_user_total_slow(user_id_param INT)
RETURNS DECIMAL(10,2) AS $$
DECLARE
    total DECIMAL(10,2);
BEGIN
    -- Counts all user orders each time
    SELECT COALESCE(SUM(total_amount), 0) INTO total
    FROM orders 
    WHERE user_id = user_id_param;
    
    RETURN total;
END;
$$ LANGUAGE plpgsql;
```

### test:
```
SELECT 
    u.user_id,
    u.username,
    get_user_total_slow(u.user_id) as total_spent
FROM users u
WHERE u.is_active = true
ORDER BY total_spent DESC
LIMIT 10;
```

### output test:
```
 user_id |  username   | total_spent
---------+-------------+-------------
  175384 | user_175384 |  1044673.34
  878092 | user_878092 |  1043331.20
  407248 | user_407248 |  1041466.21
  639301 | user_639301 |  1041455.47
  395606 | user_395606 |  1041128.07
  995316 | user_995316 |  1040497.52
     123 | user_123    |  1037672.90
  573544 | user_573544 |  1036614.62
  100105 | user_100105 |  1035034.78
  700371 | user_700371 |  1032050.46
(10 rows)
```
```
\! clear
```


## Method 2: With DP (with cache, fast):
```
-- Cache table to store results
CREATE TABLE IF NOT EXISTS user_cache (
    user_id INT PRIMARY KEY,
    total_amount DECIMAL(10,2),
    last_updated TIMESTAMP
);

-- Function: Calculate with DP (cache)
CREATE OR REPLACE FUNCTION get_user_total_fast(user_id_param INT)
RETURNS DECIMAL(10,2) AS $$
DECLARE
    cached_amount DECIMAL(10,2);
BEGIN
    -- First check if it is in the cache.
    SELECT total_amount INTO cached_amount
    FROM user_cache 
    WHERE user_id = user_id_param
    AND last_updated > CURRENT_TIMESTAMP - INTERVAL '1 hour';
    
    -- If it was in the cache, return it.
    IF cached_amount IS NOT NULL THEN
        RETURN cached_amount;
    END IF;
    
    -- If not, calculate and store in cache
    SELECT COALESCE(SUM(total_amount), 0) INTO cached_amount
    FROM orders 
    WHERE user_id = user_id_param;
    
    -- Store in cache
    INSERT INTO user_cache (user_id, total_amount, last_updated)
    VALUES (user_id_param, cached_amount, CURRENT_TIMESTAMP)
    ON CONFLICT (user_id) DO UPDATE SET
        total_amount = EXCLUDED.total_amount,
        last_updated = EXCLUDED.last_updated;
    
    RETURN cached_amount;
END;
$$ LANGUAGE plpgsql;
```

### Test speed with DP (first time - empty cache)
```
DO $$
DECLARE
    start_time TIMESTAMP;
    end_time TIMESTAMP;
    i INT;
    total DECIMAL(10,2);
BEGIN
    start_time := clock_timestamp();
    
    FOR i IN 1..10 LOOP
        SELECT get_user_total_fast((i * 10000) + 1) INTO total;
    END LOOP;
    
    end_time := clock_timestamp();
    RAISE NOTICE 'With DP (first time): % milliseconds', 
        EXTRACT(MILLISECONDS FROM (end_time - start_time));
END;
$$;
```

### output Test speed with DP (first time - empty cache):
```
NOTICE:  With DP (first time): 3.383 milliseconds
```

### Test speed with DP (subsequent times - cache filled)
```
DO $$
DECLARE
    start_time TIMESTAMP;
    end_time TIMESTAMP;
    i INT;
    total DECIMAL(10,2);
BEGIN
    start_time := clock_timestamp();
    
    FOR i IN 1..10 LOOP
        SELECT get_user_total_fast((i * 10000) + 1) INTO total;
    END LOOP;
    
    end_time := clock_timestamp();
    RAISE NOTICE 'With DP (cached): % milliseconds', 
        EXTRACT(MILLISECONDS FROM (end_time - start_time));
END;
$$;
```

### output Test speed with DP (subsequent times - cache filled):
```
NOTICE:  With DP (cached): 0.227 milliseconds
```
