# 1. Data Fundamentals

Data modeling is the foundational blueprint of data engineering. Before data can be queried, it must be structured logically. How data is modeled dictates how quickly it can be written (OLTP) and how efficiently it can be analyzed (OLAP).

!!! info "Historical Context: The Relational Era"
    In 1970, Edgar F. Codd, an IBM computer scientist, published *"A Relational Model of Data for Large Shared Data Banks"*. Before this, databases used rigid hierarchical (tree) or network models. Codd introduced the radical concept of storing data in flat tables (relations) and joining them using mathematical set theory (the foundation of SQL). This led to the creation of IBM System R and Oracle.

---

## 1.1 Relational Data Modeling (OLTP)

**Online Transaction Processing (OLTP)** systems rely on heavily normalized schemas. 

**Internal Insight:** Normalization minimizes redundancy and protects data integrity. If a user changes their email address, the database only has to update a single row in the `users` table, rather than scanning a million `orders` rows to update the email address pasted next to every purchase.

### Normal Forms
1. **1NF:** Every column must hold atomic values (no lists or JSON arrays).
2. **2NF:** Every non-key attribute must depend on the *entire* Primary Key (important for composite keys).
3. **3NF:** No non-key attribute can depend on another non-key attribute (e.g., zip code determines city, so city should be moved to a separate table).

??? example "PostgreSQL Normalization Example"
    ```sql
    -- A normalized 3NF schema for an e-commerce application
    CREATE TABLE users (
        user_id SERIAL PRIMARY KEY,
        email VARCHAR(255) UNIQUE NOT NULL,
        created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
    );

    CREATE TABLE products (
        product_id SERIAL PRIMARY KEY,
        sku VARCHAR(50) UNIQUE NOT NULL,
        price DECIMAL(10,2) NOT NULL
    );

    CREATE TABLE orders (
        order_id SERIAL PRIMARY KEY,
        user_id INT REFERENCES users(user_id),
        order_date TIMESTAMP DEFAULT CURRENT_TIMESTAMP
    );

    -- The associative entity resolving the Many-to-Many relationship
    CREATE TABLE order_items (
        order_id INT REFERENCES orders(order_id),
        product_id INT REFERENCES products(product_id),
        quantity INT DEFAULT 1,
        PRIMARY KEY (order_id, product_id)
    );
    ```

---

## 1.2 Dimensional Modeling (OLAP)

**Online Analytical Processing (OLAP)** systems reject extreme normalization because traversing 10 `JOIN`s to calculate total revenue is incredibly slow.

!!! info "Historical Context: The Kimball vs Inmon Debate"
    In the 1990s, the Data Warehousing world split. **Bill Inmon** advocated for a top-down, strictly normalized enterprise data warehouse. **Ralph Kimball** (author of *The Data Warehouse Toolkit*, 1996) advocated for a bottom-up approach using **Star Schemas** (Facts and Dimensions) that were explicitly denormalized for query performance and human readability. Kimball's dimensional modeling won out and remains the standard today.

### The Star Schema
1. **Fact Tables:** Massive tables containing measurable events (e.g., `fact_orders`). They contain foreign keys and numbers (revenue, quantity).
2. **Dimension Tables:** Smaller, wider tables containing descriptive attributes (e.g., `dim_user`, `dim_date`, `dim_product`).

??? example "BigQuery Star Schema Example"
    ```sql
    -- A denormalized Dimension table (violates 3NF for query speed)
    CREATE TABLE dim_location (
        location_id INT PRIMARY KEY,
        zip_code VARCHAR(10),
        city VARCHAR(50),
        state VARCHAR(50),      -- Redundant, but fast!
        country VARCHAR(50)
    );

    -- The Fact table containing the numeric metrics
    CREATE TABLE fact_sales (
        sale_id INT PRIMARY KEY,
        date_id INT REFERENCES dim_date(date_id),
        product_id INT REFERENCES dim_product(product_id),
        location_id INT REFERENCES dim_location(location_id),
        revenue DECIMAL(10,2),
        discount DECIMAL(10,2)
    );
    ```

---

## 1.3 NoSQL Data Modeling

When scaling beyond a single server, NoSQL databases (like Cassandra or DynamoDB) require a complete inversion of traditional modeling rules.

**Internal Insight:** In a massively distributed cluster, `JOIN`s across nodes require network network hops, destroying latency. NoSQL solves this by aggressively **Denormalizing** data. You design the schema specifically *for the query* you intend to run, duplicating data without hesitation.

??? example "Cassandra Application-Driven Schema Design"
    If you need to query a user's latest orders, you do not join `users` and `orders`. You create a table explicitly for that view.
    
    ```sql
    -- Duplicating the user's email inside the orders table!
    CREATE TABLE recent_orders_by_user (
        user_id UUID,
        user_email TEXT,
        order_id UUID,
        total_price DECIMAL,
        order_date TIMESTAMP,
        PRIMARY KEY ((user_id), order_date)
    ) WITH CLUSTERING ORDER BY (order_date DESC);
    ```

---

!!! abstract "References & Papers"
    - **A Relational Model of Data for Large Shared Data Banks** (Codd, CACM 1970). The paper that invented SQL.
    - **The Data Warehouse Toolkit** (Kimball, 1996). The definitive guide to Dimensional Modeling.
    - **Designing Data-Intensive Applications** - Chapter 2 (Data Models and Query Languages) details the tension between Relational, Document, and Graph data models.
