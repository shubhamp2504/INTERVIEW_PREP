# 🏗️ System Design — JPMorgan & Banking Interview Problems

> System design questions from JPMorgan Chase and banking/finance interviews.

---

## Q1. Explain the Design of Your Recent Project

### 📝 One-Liner
Walk through your project's architecture, key design decisions, and trade-offs.

### 🔑 Quick Answer
Structure your answer: Context → Architecture → Key Components → Data Flow → Challenges → Trade-offs. *(project ka overview do, architecture batao, challenges aur trade-offs discuss karo)*

### 📖 How It Works
Use this template to prepare:
1. **Context**: What does the system do? Scale? Users?
2. **Architecture**: Monolith/microservices? Sync/async? Message queues?
3. **Key Components**: List 3-5 main services and their roles
4. **Data Flow**: Request lifecycle end-to-end
5. **Database Design**: Schema choices, why SQL vs NoSQL
6. **Challenges**: Scaling bottlenecks, reliability issues you solved
7. **Trade-offs**: Why you chose X over Y

### 🗣️ Answering Approach
"Our system is a [domain] platform handling [scale]. It's built as microservices communicating via REST and Kafka. Let me walk through the key components: [Service A] handles..., [Service B] processes... The main challenge was [problem], which we solved by [solution]. A key trade-off was choosing [X] over [Y] because..."

### ⚠️ Pitfalls / Gotchas
- Don't just describe — explain WHY each decision was made
- Be ready for deep-dive follow-ups on any component
- Know the numbers: requests/sec, latency p99, data volume *(numbers yaad rakho — interviewer impress hota hai)*
- Don't reveal proprietary/confidential details

### ⚡ Remember
- Prepare 2-3 projects with different complexity levels
- Always mention: scale, trade-offs, alternatives considered
- Draw a diagram (even verbally describe the diagram)
- Connect to interviewing company's domain when possible

---

## Q2. Database Design for Ride-Sharing

### 📝 One-Liner
Design a database schema supporting riders, drivers, trips, payments, and ratings.

### 🔑 Quick Answer
Core tables: Users (riders/drivers), Trips (with state machine), Payments, Ratings. Geospatial index for driver location. *(Users, Trips, Payments tables — location ke liye geospatial index)*

### 📖 How It Works

**Core Schema**:
```sql
-- Users (both riders and drivers)
CREATE TABLE users (
    id BIGINT PRIMARY KEY,
    type ENUM('RIDER', 'DRIVER'),
    name VARCHAR(100),
    phone VARCHAR(15) UNIQUE,
    rating DECIMAL(3,2),
    created_at TIMESTAMP
);

-- Driver Location (separate for performance)
CREATE TABLE driver_locations (
    driver_id BIGINT PRIMARY KEY,
    location POINT NOT NULL,  -- PostGIS geospatial
    status ENUM('AVAILABLE', 'ON_TRIP', 'OFFLINE'),
    updated_at TIMESTAMP,
    SPATIAL INDEX(location)
);

-- Trips
CREATE TABLE trips (
    id BIGINT PRIMARY KEY,
    rider_id BIGINT REFERENCES users(id),
    driver_id BIGINT REFERENCES users(id),
    pickup POINT, dropoff POINT,
    status ENUM('REQUESTED','MATCHED','IN_PROGRESS','COMPLETED','CANCELLED'),
    fare DECIMAL(10,2),
    distance_km DECIMAL(8,2),
    started_at TIMESTAMP, completed_at TIMESTAMP
);

-- Payments
CREATE TABLE payments (
    id BIGINT PRIMARY KEY,
    trip_id BIGINT REFERENCES trips(id),
    amount DECIMAL(10,2),
    method ENUM('CARD','WALLET','CASH'),
    status ENUM('PENDING','SUCCESS','FAILED','REFUNDED'),
    idempotency_key VARCHAR(64) UNIQUE
);
```

### 🗣️ Answering Approach
"I'd separate driver locations into a dedicated table with a spatial index since it's updated every few seconds. The trips table uses a state machine enum. Payments have an idempotency key to prevent double charges. For read-heavy queries like 'find nearby drivers,' I'd use PostGIS spatial queries."

### ⚡ Remember
- Separate location table: write-heavy, updated every 5 seconds
- PostGIS or Geohash index for spatial queries
- Trip state machine: REQUESTED → MATCHED → IN_PROGRESS → COMPLETED
- Index on `(rider_id, created_at)` for trip history queries

---

## Q3. Design a Data Warehouse for Online Retailer

### 📝 One-Liner
Design an analytical data warehouse for an e-commerce company to support reporting and BI.

### 🔑 Quick Answer
Star schema with fact tables (orders, clicks, inventory) and dimension tables (products, customers, time, location). ETL pipeline from OLTP to OLAP. *(star schema — facts aur dimensions, ETL se data load)*

### 📖 How It Works
1. **Source Systems**: OLTP databases (orders, users, products, inventory)
2. **ETL Pipeline**: Extract from sources → Transform (clean, aggregate) → Load into DW
3. **Schema Design**: Star schema
   - **Fact Tables**: `fact_orders`, `fact_page_views`, `fact_inventory`
   - **Dimension Tables**: `dim_product`, `dim_customer`, `dim_time`, `dim_location`
4. **Query Patterns**: Aggregations, trends, cohort analysis
5. **Tools**: Spark/Airflow for ETL, Redshift/BigQuery/Snowflake for DW

### 💻 Code
```sql
-- Star Schema Example
CREATE TABLE fact_orders (
    order_id BIGINT,
    customer_key INT REFERENCES dim_customer(key),
    product_key INT REFERENCES dim_product(key),
    time_key INT REFERENCES dim_time(key),
    location_key INT REFERENCES dim_location(key),
    quantity INT,
    total_amount DECIMAL(12,2),
    discount DECIMAL(10,2)
);

CREATE TABLE dim_product (
    key INT PRIMARY KEY,  -- surrogate key
    product_id BIGINT,    -- natural key
    name VARCHAR(200),
    category VARCHAR(100),
    brand VARCHAR(100),
    effective_from DATE,  -- SCD Type 2
    effective_to DATE
);

-- Common Query: Sales by category per month
SELECT dp.category, dt.month_name, SUM(fo.total_amount) as total_sales
FROM fact_orders fo
JOIN dim_product dp ON fo.product_key = dp.key
JOIN dim_time dt ON fo.time_key = dt.key
GROUP BY dp.category, dt.month_name
ORDER BY total_sales DESC;
```

### ⚡ Remember
- Star schema: one fact table surrounded by dimensions
- Snowflake schema: normalized dimensions (more joins, less space)
- SCD Type 2: track historical changes in dimensions
- Partitioning: by date for time-series queries

---

## Q4. Design a News Aggregator

### 📝 One-Liner
A system that crawls, categorizes, and serves news articles from multiple sources with personalized feeds.

### 🔑 Quick Answer
Crawler fetches articles → NLP pipeline categorizes → ranked feed per user based on interests and trending topics. *(crawl karo, categorize karo, user ko personalized feed do)*

### 📖 How It Works
1. **Crawler**: Scheduled jobs fetch RSS/API from news sources → dedup by URL hash
2. **Processing**: NLP pipeline: extract text → categorize (politics, tech, sports) → sentiment
3. **Storage**: Articles in Elasticsearch (search) + Cassandra (feed)
4. **Ranking**: Score = f(recency, category_affinity, source_trust, trending)
5. **Feed**: Pre-computed per-user or real-time merge of subscribed categories
6. **Trending**: Sliding window count of articles per topic

### 🗣️ Answering Approach
"I'd build a multi-stage pipeline. Crawlers fetch articles from RSS feeds on a schedule. A processing pipeline extracts content, categorizes using ML, and detects duplicates. Articles are indexed in Elasticsearch for search and Cassandra for feed serving. The feed is ranked by a combination of recency, user interests, and trending score."

### ⚡ Remember
- Dedup: hash(title + source) to catch reprints
- Freshness: recent articles ranked higher
- Politeness: respect robots.txt, rate limit crawls
- Personalization: collaborative filtering + content-based

---
