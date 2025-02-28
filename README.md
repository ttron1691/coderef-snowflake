# coderef-snowflake

# Snowflake SQL Reference Card

## Table of Contents
- [Data Definition Language (DDL)](#data-definition-language-ddl)
- [Data Manipulation Language (DML)](#data-manipulation-language-dml)
- [Query Optimization](#query-optimization)
- [Snowflake-Specific Features](#snowflake-specific-features)
- [Security Best Practices](#security-best-practices)
- [Performance Tips](#performance-tips)
- [Cost Optimization](#cost-optimization)

## Data Definition Language (DDL)

### Database Objects

#### Create Database
```sql
CREATE DATABASE my_database;
```

#### Create Schema
```sql
CREATE SCHEMA my_database.my_schema;
```

#### Create Table
```sql
CREATE TABLE my_database.my_schema.my_table (
  id NUMBER PRIMARY KEY,
  name VARCHAR(255) NOT NULL,
  created_at TIMESTAMP_NTZ DEFAULT CURRENT_TIMESTAMP()
);
```

#### Create Table with Clustering Key
```sql
CREATE TABLE my_database.my_schema.events (
  event_id NUMBER,
  event_date DATE,
  user_id NUMBER,
  event_type VARCHAR
)
CLUSTER BY (event_date, user_id);
```

#### Create View
```sql
CREATE OR REPLACE VIEW my_database.my_schema.active_users AS
  SELECT user_id, name, last_login_date
  FROM my_database.my_schema.users
  WHERE status = 'active';
```

#### Create Materialized View
```sql
CREATE OR REPLACE MATERIALIZED VIEW my_database.my_schema.daily_sales AS
  SELECT 
    date_trunc('day', sale_timestamp) AS sale_date,
    SUM(amount) AS total_amount,
    COUNT(*) AS transaction_count
  FROM my_database.my_schema.sales
  GROUP BY 1;
```

#### Create External Table
```sql
CREATE EXTERNAL TABLE my_database.my_schema.ext_customer_data (
  customer_id NUMBER AS (VALUE:c_id::NUMBER),
  customer_name VARCHAR AS (VALUE:c_name::VARCHAR),
  customer_email VARCHAR AS (VALUE:c_email::VARCHAR)
)
WITH LOCATION = @my_database.my_schema.my_stage/customer_data/
FILE_FORMAT = (TYPE = 'JSON');
```

## Data Manipulation Language (DML)

### Basic Queries

#### Select Data
```sql
SELECT * FROM my_database.my_schema.customers
WHERE region = 'Northeast'
ORDER BY signup_date DESC
LIMIT 100;
```

#### Insert Data
```sql
INSERT INTO my_database.my_schema.customers (customer_id, name, email)
VALUES 
  (1001, 'Alice Smith', 'alice@example.com'),
  (1002, 'Bob Johnson', 'bob@example.com');
```

#### Update Data
```sql
UPDATE my_database.my_schema.customers
SET status = 'inactive', last_updated = CURRENT_TIMESTAMP()
WHERE last_login_date < DATEADD('day', -90, CURRENT_DATE());
```

#### Delete Data
```sql
DELETE FROM my_database.my_schema.customers
WHERE status = 'deleted' 
  AND last_updated < DATEADD('month', -12, CURRENT_DATE());
```

### Advanced Queries

#### CTE (Common Table Expression)
```sql
WITH monthly_revenue AS (
  SELECT 
    date_trunc('month', order_date) AS month,
    customer_id,
    SUM(order_amount) AS revenue
  FROM my_database.my_schema.orders
  WHERE order_date >= DATEADD('year', -1, CURRENT_DATE())
  GROUP BY 1, 2
)
SELECT 
  month,
  COUNT(DISTINCT customer_id) AS customers,
  SUM(revenue) AS total_revenue,
  AVG(revenue) AS avg_revenue_per_customer
FROM monthly_revenue
GROUP BY 1
ORDER BY 1;
```

#### Window Functions
```sql
SELECT 
  order_id,
  customer_id,
  order_amount,
  order_date,
  SUM(order_amount) OVER (PARTITION BY customer_id ORDER BY order_date) AS customer_running_total,
  AVG(order_amount) OVER (PARTITION BY customer_id) AS customer_avg_order,
  RANK() OVER (PARTITION BY customer_id ORDER BY order_amount DESC) AS order_rank
FROM my_database.my_schema.orders;
```

#### Pivot Tables
```sql
SELECT * FROM (
  SELECT product_category, order_month, sales_amount
  FROM my_database.my_schema.sales
) 
PIVOT (
  SUM(sales_amount) 
  FOR order_month IN ('2024-01', '2024-02', '2024-03', '2024-04')
) AS p
ORDER BY product_category;
```

## Query Optimization

### Best Practices

#### Filter Early
```sql
-- Good: Apply filters early
SELECT o.order_id, o.order_date, c.customer_name
FROM orders o
JOIN customers c ON o.customer_id = c.customer_id
WHERE o.order_date >= DATEADD('month', -3, CURRENT_DATE());

-- Less efficient: Filter later
SELECT o.order_id, o.order_date, c.customer_name
FROM orders o
JOIN customers c ON o.customer_id = c.customer_id
WHERE o.order_date >= DATEADD('month', -3, CURRENT_DATE());
```

#### Avoid SELECT *
```sql
-- Good: Select only needed columns
SELECT customer_id, order_id, order_amount
FROM orders
WHERE order_date > '2024-01-01';

-- Less efficient: Select all columns
SELECT *
FROM orders
WHERE order_date > '2024-01-01';
```

#### Use Semi-joins
```sql
-- Good: Use EXISTS or IN
SELECT customer_id, customer_name
FROM customers
WHERE EXISTS (
  SELECT 1 FROM orders 
  WHERE orders.customer_id = customers.customer_id
  AND order_date > '2024-01-01'
);

-- Alternative approach with semi-join
SELECT DISTINCT c.customer_id, c.customer_name
FROM customers c
JOIN orders o ON c.customer_id = o.customer_id
WHERE o.order_date > '2024-01-01';
```

## Snowflake-Specific Features

### Zero-Copy Cloning
```sql
-- Clone a table
CREATE OR REPLACE TABLE my_database.my_schema.customers_backup
CLONE my_database.my_schema.customers;

-- Clone a database
CREATE DATABASE my_database_dev
CLONE my_database;
```

### Time Travel
```sql
-- Query data as it existed 2 hours ago
SELECT * FROM my_database.my_schema.customers
AT(OFFSET => -60*60*2);

-- Query data as it existed at a specific timestamp
SELECT * FROM my_database.my_schema.customers
AT(TIMESTAMP => '2024-02-26 12:00:00'::TIMESTAMP_NTZ);

-- Restore a table to a previous state
CREATE OR REPLACE TABLE my_database.my_schema.customers
CLONE my_database.my_schema.customers BEFORE (STATEMENT => '019a2a82-b170-44c6-af3f-f5c4a99dbf9e');
```

### Semi-Structured Data
```sql
-- Query JSON data
SELECT 
  metadata:customer.id::NUMBER AS customer_id,
  metadata:customer.name::VARCHAR AS customer_name,
  metadata:device.type::VARCHAR AS device_type,
  event_type
FROM my_database.my_schema.events
WHERE metadata:customer.segment::VARCHAR = 'premium';

-- Flatten array of objects
SELECT 
  t.order_id,
  f.value:product_id::NUMBER AS product_id,
  f.value:quantity::NUMBER AS quantity,
  f.value:price::DECIMAL(10,2) AS price
FROM my_database.my_schema.orders t,
LATERAL FLATTEN(input => t.line_items) f;
```

### Secure Data Sharing
```sql
-- Create a share
CREATE OR REPLACE SHARE my_customer_share;

-- Grant usage on database to the share
GRANT USAGE ON DATABASE my_database TO SHARE my_customer_share;
GRANT USAGE ON SCHEMA my_database.my_schema TO SHARE my_customer_share;
GRANT SELECT ON TABLE my_database.my_schema.aggregated_metrics TO SHARE my_customer_share;

-- Add account to share
ALTER SHARE my_customer_share ADD ACCOUNTS = partner_account;
```

## Security Best Practices

### Role-Based Access Control
```sql
-- Create roles with different privilege levels
CREATE ROLE data_analyst;
CREATE ROLE data_scientist;
CREATE ROLE data_engineer;

-- Grant privileges
GRANT USAGE ON WAREHOUSE analytics_wh TO ROLE data_analyst;
GRANT USAGE ON DATABASE analytics_db TO ROLE data_analyst;
GRANT USAGE ON SCHEMA analytics_db.public TO ROLE data_analyst;
GRANT SELECT ON ALL TABLES IN SCHEMA analytics_db.public TO ROLE data_analyst;

-- Grant additional privileges to higher roles
GRANT ROLE data_analyst TO ROLE data_scientist;
GRANT CREATE TABLE ON SCHEMA analytics_db.public TO ROLE data_scientist;
```

### Column-Level Security
```sql
-- Create a masking policy for PII
CREATE OR REPLACE MASKING POLICY email_mask AS
  (val STRING) RETURNS STRING ->
    CASE
      WHEN CURRENT_ROLE() IN ('ADMIN', 'COMPLIANCE') THEN val
      ELSE REGEXP_REPLACE(val, '^(.)(.*)(.)(@.*)$', '\\1***\\3\\4')
    END;

-- Apply the masking policy to a column
ALTER TABLE my_database.my_schema.customers 
MODIFY COLUMN email SET MASKING POLICY email_mask;
```

### Row-Level Security
```sql
-- Create a row access policy
CREATE OR REPLACE ROW ACCESS POLICY region_access AS
  (region VARCHAR) RETURNS BOOLEAN ->
    region IN (SELECT allowed_region FROM access_control WHERE role_name = CURRENT_ROLE());

-- Apply the row access policy to a table
ALTER TABLE my_database.my_schema.customers
ADD ROW ACCESS POLICY region_access ON (region);
```

## Performance Tips

### Warehouse Sizing
```sql
-- Create appropriately sized warehouses for different workloads
CREATE WAREHOUSE reporting_wh 
  WITH WAREHOUSE_SIZE = 'LARGE'
  AUTO_SUSPEND = 300
  AUTO_RESUME = TRUE
  INITIALLY_SUSPENDED = TRUE;

CREATE WAREHOUSE etl_wh 
  WITH WAREHOUSE_SIZE = 'X-LARGE'
  AUTO_SUSPEND = 600
  AUTO_RESUME = TRUE
  MIN_CLUSTER_COUNT = 1
  MAX_CLUSTER_COUNT = 5
  SCALING_POLICY = 'STANDARD';
```

### Result Caching
```sql
-- Enable result caching at session level
ALTER SESSION SET USE_CACHED_RESULT = TRUE;

-- Force recomputation by adding a unique comment
SELECT /* 202402271212 */ 
  date_trunc('month', order_date) AS month,
  SUM(order_amount) AS total_sales
FROM my_database.my_schema.orders
GROUP BY 1
ORDER BY 1;
```

### Clustering Keys
```sql
-- Analyze clustering depth
SELECT * FROM TABLE(SYSTEM$CLUSTERING_DEPTH(
  'my_database.my_schema.large_table', '(event_date, customer_id)'));

-- Recluster a table
ALTER TABLE my_database.my_schema.large_table RECLUSTER;
```

## Cost Optimization

### Resource Monitors
```sql
-- Create a resource monitor
CREATE OR REPLACE RESOURCE MONITOR monthly_limit
  WITH CREDIT_QUOTA = 5000
  FREQUENCY = MONTHLY
  START_TIMESTAMP = IMMEDIATELY
  TRIGGERS
    ON 80 PERCENT DO NOTIFY
    ON 90 PERCENT DO NOTIFY
    ON 100 PERCENT DO SUSPEND;
    
-- Assign resource monitor to a warehouse
ALTER WAREHOUSE reporting_wh SET RESOURCE_MONITOR = monthly_limit;
```

### Query Profiling
```sql
-- Examine query profile
SELECT * FROM TABLE(INFORMATION_SCHEMA.QUERY_HISTORY())
WHERE QUERY_TEXT LIKE '%customer_order_summary%'
ORDER BY START_TIME DESC
LIMIT 10;

-- Get detailed execution statistics
SELECT * FROM TABLE(INFORMATION_SCHEMA.QUERY_HISTORY_BY_ID('019a2a82-b170-44c6-af3f-f5c4a99dbf9e'));
```

### Storage Optimization
```sql
-- Check table storage metrics
SELECT * FROM TABLE(INFORMATION_SCHEMA.TABLE_STORAGE_METRICS('my_database.my_schema.large_table'));

-- Optimize storage with clustering
ALTER TABLE my_database.my_schema.large_table CLUSTER BY (date_column, category);

-- Set appropriate retention periods
ALTER DATABASE my_database
SET DATA_RETENTION_TIME_IN_DAYS = 7;
```

