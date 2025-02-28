# Analytics and Reporting

## OLAP Operations
### Cube and Rollup
```sql
-- Create sample sales data
CREATE TABLE sales_data (
    date_id DATE,
    product_id INTEGER,
    region_id INTEGER,
    sales_amount NUMERIC,
    units_sold INTEGER
);

-- CUBE example
SELECT 
    COALESCE(date_trunc('month', date_id)::TEXT, 'All Dates') as month,
    COALESCE(product_id::TEXT, 'All Products') as product,
    COALESCE(region_id::TEXT, 'All Regions') as region,
    SUM(sales_amount) as total_sales
FROM sales_data
GROUP BY CUBE(date_trunc('month', date_id), product_id, region_id);

-- ROLLUP example
SELECT 
    date_trunc('year', date_id) as year,
    date_trunc('month', date_id) as month,
    region_id,
    SUM(sales_amount) as total_sales
FROM sales_data
GROUP BY ROLLUP(
    date_trunc('year', date_id),
    date_trunc('month', date_id),
    region_id
);
```

### Window Functions for Analytics
```sql
-- Moving averages
SELECT 
    date_id,
    region_id,
    sales_amount,
    AVG(sales_amount) OVER (
        PARTITION BY region_id 
        ORDER BY date_id 
        ROWS BETWEEN 6 PRECEDING AND CURRENT ROW
    ) as weekly_moving_avg
FROM sales_data;

-- Year-over-year comparison
WITH monthly_sales AS (
    SELECT 
        date_trunc('month', date_id) as month,
        SUM(sales_amount) as sales
    FROM sales_data
    GROUP BY 1
)
SELECT 
    month,
    sales,
    LAG(sales, 12) OVER (ORDER BY month) as last_year_sales,
    (sales - LAG(sales, 12) OVER (ORDER BY month)) / 
        LAG(sales, 12) OVER (ORDER BY month) * 100 as yoy_growth
FROM monthly_sales;
```

## Materialized Views
### Creating Materialized Views
```sql
-- Create materialized view for daily sales
CREATE MATERIALIZED VIEW daily_sales_summary AS
SELECT 
    date_id,
    region_id,
    SUM(sales_amount) as total_sales,
    COUNT(DISTINCT product_id) as products_sold,
    SUM(units_sold) as total_units
FROM sales_data
GROUP BY date_id, region_id
WITH DATA;

-- Create index on materialized view
CREATE INDEX idx_daily_sales_date 
ON daily_sales_summary(date_id);

-- Refresh materialized view
REFRESH MATERIALIZED VIEW daily_sales_summary;
```

### Incremental Refresh Strategy
```sql
-- Create materialized view with last_updated timestamp
CREATE MATERIALIZED VIEW sales_summary_incremental AS
SELECT 
    date_id,
    region_id,
    SUM(sales_amount) as total_sales,
    now() as last_updated
FROM sales_data
GROUP BY date_id, region_id
WITH DATA;

-- Create function for incremental refresh
CREATE OR REPLACE FUNCTION refresh_sales_summary()
RETURNS void AS $$
DECLARE
    v_last_updated TIMESTAMP;
BEGIN
    -- Get last update time
    SELECT last_updated INTO v_last_updated
    FROM sales_summary_incremental
    LIMIT 1;
    
    -- Create temporary table for new data
    CREATE TEMP TABLE new_sales_summary AS
    SELECT 
        date_id,
        region_id,
        SUM(sales_amount) as total_sales,
        now() as last_updated
    FROM sales_data
    WHERE updated_at > v_last_updated
    GROUP BY date_id, region_id;
    
    -- Update existing records
    UPDATE sales_summary_incremental s
    SET total_sales = n.total_sales,
        last_updated = n.last_updated
    FROM new_sales_summary n
    WHERE s.date_id = n.date_id
    AND s.region_id = n.region_id;
    
    -- Insert new records
    INSERT INTO sales_summary_incremental
    SELECT * FROM new_sales_summary n
    WHERE NOT EXISTS (
        SELECT 1 FROM sales_summary_incremental s
        WHERE s.date_id = n.date_id
        AND s.region_id = n.region_id
    );
    
    DROP TABLE new_sales_summary;
END;
$$ LANGUAGE plpgsql;
```

## Analytical Functions
### Advanced Analytics
```sql
-- Percentile calculations
SELECT 
    product_id,
    sales_amount,
    PERCENT_RANK() OVER (ORDER BY sales_amount) as percentile,
    NTILE(4) OVER (ORDER BY sales_amount) as quartile
FROM sales_data;

-- Running totals and moving calculations
SELECT 
    date_id,
    sales_amount,
    SUM(sales_amount) OVER (ORDER BY date_id) as running_total,
    AVG(sales_amount) OVER (
        ORDER BY date_id 
        ROWS BETWEEN 30 PRECEDING AND CURRENT ROW
    ) as monthly_moving_avg
FROM sales_data;
```

## Report Optimization
### Performance Tuning
```sql
-- Create summary tables
CREATE TABLE sales_daily_summary AS
SELECT 
    date_id,
    region_id,
    product_id,
    SUM(sales_amount) as total_sales,
    COUNT(*) as transaction_count,
    SUM(units_sold) as units_sold
FROM sales_data
GROUP BY date_id, region_id, product_id;

-- Create appropriate indexes
CREATE INDEX idx_sales_summary_date 
ON sales_daily_summary(date_id);
CREATE INDEX idx_sales_summary_region 
ON sales_daily_summary(region_id);
```

### Automated Report Generation
```sql
-- Create report function
CREATE OR REPLACE FUNCTION generate_sales_report(
    start_date DATE,
    end_date DATE
)
RETURNS TABLE (
    period TEXT,
    region_name TEXT,
    total_sales NUMERIC,
    growth_rate NUMERIC
) AS $$
BEGIN
    RETURN QUERY
    WITH monthly_sales AS (
        SELECT 
            date_trunc('month', s.date_id) as period,
            r.region_name,
            SUM(s.sales_amount) as sales
        FROM sales_data s
        JOIN regions r ON s.region_id = r.region_id
        WHERE s.date_id BETWEEN start_date AND end_date
        GROUP BY 1, 2
    )
    SELECT 
        period::TEXT,
        region_name,
        sales as total_sales,
        (sales - LAG(sales) OVER (PARTITION BY region_name ORDER BY period)) / 
            LAG(sales) OVER (PARTITION BY region_name ORDER BY period) * 100 
            as growth_rate
    FROM monthly_sales
    ORDER BY period, region_name;
END;
$$ LANGUAGE plpgsql;
```

## Best Practices
1. Performance
   - Use materialized views for complex calculations
   - Implement incremental refresh strategies
   - Create appropriate indexes
   - Partition large tables

2. Data Organization
   - Create summary tables
   - Use effective naming conventions
   - Document calculations
   - Maintain data lineage

3. Report Design
   - Consider user requirements
   - Optimize for common queries
   - Plan for scalability
   - Implement caching strategies

## Related Topics
- [[Query Optimization]]
- [[Performance Tuning]]
- [[Materialized Views]]
- [[Partitioning Strategies]]
