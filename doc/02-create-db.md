
### Login to PostgreSQL:
```
sudo -i -u postgres
psql
```

### create db:
```
CREATE DATABASE ecommerce_db;
\c ecommerce_db
```

### create table
```
CREATE TABLE users (
    user_id SERIAL PRIMARY KEY,
    username VARCHAR(50) NOT NULL,
    email VARCHAR(100) NOT NULL,
    city VARCHAR(50),
    country VARCHAR(50),
    registration_date DATE,
    last_login TIMESTAMP,
    is_active BOOLEAN,
    account_type VARCHAR(20),
    total_purchases DECIMAL(10,2)
);
```

```
\d users
\d+ users
\dt
```
```
\! clear
```

### create indexs:
```
CREATE INDEX idx_users_city ON users(city);
CREATE INDEX idx_users_country ON users(country);
CREATE INDEX idx_users_active ON users(is_active);
CREATE INDEX idx_users_account_type ON users(account_type);
```
```
\di
```
```
\! clear
```

### Populating a table with 1 million records
```
INSERT INTO users (username, email, city, country, registration_date, last_login, is_active, account_type, total_purchases)
SELECT 
    'user_' || generate_series(1, 1000000),
    'user_' || generate_series(1, 1000000) || '@example.com',
    (ARRAY['Tehran', 'Mashhad', 'Isfahan', 'Shiraz', 'Tabriz', 'Karaj', 'Qom', 'Ahvaz'])[floor(random() * 8 + 1)],
    (ARRAY['Iran', 'USA', 'Canada', 'UK', 'Germany', 'Turkey', 'UAE', 'Japan'])[floor(random() * 8 + 1)],
    CURRENT_DATE - (random() * 3650)::INT,
    CURRENT_TIMESTAMP - (random() * INTERVAL '365 days'),
    random() > 0.3,
    (ARRAY['free', 'basic', 'premium', 'enterprise'])[floor(random() * 4 + 1)],
    random() * 10000
;
```
```
SELECT count(*) FROM users;
```

