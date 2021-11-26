# Up and Running: Setting Up Your Data Engineering Infrastructure on AWS
The completely free E-Book for setting up and running a Data Engineering stack on AWS and Snowflake.

NOTE: This book is currently incomplete. If you find errors or would like to fill in the gaps, read the [Contributions section](https://github.com/Nunie123/data_engineering_on_aws#user-content-contributions).

## Table of Contents
[Introduction](https://github.com/Nunie123/data_engineering_on_aws) <br>
[Chapter 1: Setting Up Your Accounts](https://github.com/Nunie123/data_engineering_on_aws/blob/main/01_accounts.md) <br>
**Chapter 2: Data Warehousing with Snowflake** (You are here) <br>
Chapter 3: Infrastructure as Code with Terraform and Terragrunt <br>
Chapter 4: Streaming Data with Snowpipe and AWS Kinesis <br>
Chapter 5: Orchestrating Pipelines with Airflow and AWS MWAA <br>
Chapter 6: Transforming Data with dbt <br>
Chapter 7: Processing Data with AWS Batch <br>
Chapter 8: Event-Driven Pipelines with AWS Lambda and SNS <br>
Chapter 9: Deployment Pipelines with GitHub Actions <br>
Chapter 10: Alerting with AWS CloudWatch and Airflow

---

# Chapter 2: Data Warehousing with Snowflake

Data Engineers are primarily interested in moving data around and deciding how data should be stored. Frequently the answer to the question "Where should we store the data?" is "In our data warehouse." Since it doesn't make much sense to move data around if we have no where to put it, the first step in creating our infrastructure is to build our data warehouse.

We are going to be using Snowflake for our data warehouse, but there are other good options out there, such as Google's BigQuery and AWS' Redshift.

## The Scenario

For the remainder of this book we are going to assume we work for an e-commerce company and are building their data infrastructure from the ground up. This company collects data through its web application, has a payment processor that collect data on its behalf, and buys data from third party vendors.

We will spend the rest of this book gathering this data and making it available to our company's analytics team.

## ELT vs. ETL

It used to be that the term "data pipeline" was synonymous with "ETL". "ETL" being an acronym for "Extract", "Transform", and "Load". First we would gather the data from the source system, then we would transform the data to how wanted it to look in our warehouse, and then finally we would load it into our warehouse. However, with the rise of powerful and (relatively) cheap data warehouse technology, it is common to see Data Engineers use an "ELT" design, instead.

In ELT, the data is extracted from the source system, then it is loaded, as-is, into the data warehouse. Finally, once the data is in the warehouse we will use the computing power of the warehouse to transform the data into a form we want to expose to our users.

Both approaches are reasonable choices, depending on your circumstances, but in this book we are going to take an ELT approach.

## Transactional vs. Analytical Databases

Databases are not just the purview of Data Engineers, they are critical to any moderately complex computer application. But most applications, such as the web application of our fictional e-commerce company, use a transactional database. These are databases designed to efficiently run transactions on a table, such as adding a user record or looking up the stock count from an inventory table.

One of the main differences is that transactional databases are concerned with adding, updating, and reading individual rows. Analytical databases are often more concerned about aggregating all the rows, such as showing the count of all products sold last month. This is why you might also here analytical databases being referred to as "columnar" databases, because the are optimized for aggregating columns instead of retrieving rows.

Another difference between transactional and analytical databases is how the data is modeled (e.g. what tables are in the database and what fields are on the tables). A transactional database is modeled in a way that is optimized for interacting with the application it supports. Analytical databases are modeled to support analysis of that data. So while both types of database may contain the same data, it will be organized differently.

## Our Source Schema

Let's assume we are streaming data from our company's application database, which is MySQL 8.0 running on AWS RDS (we will set up this streaming pipeline in Chapter 4). As we are following an ELT design, we need to create tables with a similar design as exists in the source system, so the data can be loaded without transformation.

Below is an Entity Relation Diagram (ERD) describing the schema of the source system.

![ERD for an e-commerce company](/images/de_on_aws_source_erd.drawio.png)

We have seven tables from the source system that we are copying over, so we need to set up tables in our warehouse to receive that data. In the Snowflake web console navigate to the "Worksheets" tab. From there, execute the following statements:
``` SQL
CREATE DATABASE landing;

CREATE SCHEMA landing.web_application_db;

CREATE TABLE landing.web_application_db.products (
    id                      INT,
    name                    VARCHAR,
    price                   NUMERIC,
    category                VARCHAR,
    snowflake_uploaded_at   TIMESTAMP_TZ
);

CREATE TABLE landing.web_application_db.customers (
    id                      INT,
    name                    VARCHAR,
    email                   VARCHAR,
    address                 VARCHAR,
    snowflake_uploaded_at   TIMESTAMP_TZ
);

CREATE TABLE landing.web_application_db.suppliers (
    id                      INT,
    name                    VARCHAR,
    address                 NUMERIC,
    phone                   VARCHAR,
    snowflake_uploaded_at   TIMESTAMP_TZ
);

CREATE TABLE landing.web_application_db.inventory (
    id                          INT,
    product_id                  VARCHAR,
    product_count               INT,
    snowflake_uploaded_at       TIMESTAMP_TZ
);

CREATE TABLE landing.web_application_db.product_suppliers (
    id                      INT,
    product_id              INT,
    supplier_id             INT,
    snowflake_uploaded_at   TIMESTAMP_TZ
);

CREATE TABLE landing.web_application_db.sales (
    id                      INT,
    customer_id             INT,
    product_id              INT,
    sales_date              TIMESTAMP_TZ,
    snowflake_uploaded_at   TIMESTAMP_TZ
);

CREATE TABLE landing.web_application_db.user_sessions (
    id                      INT,
    customer_id             INT,
    page                    VARCHAR,
    visited_at              TIMESTAMP_TZ,
    snowflake_uploaded_at   TIMESTAMP_TZ
);
```

This code establishes a database where we'll put all our raw data (`LANDING`), a schema for data from our company's web application database (`WEB_APPLICATION_DB`), and seven tables to hold all the application data. We've also added a `snowflake_uploaded_at` field so we have a timestamp of when the data was uploaded.

## Our Data Warehouse Schema

The (simplified) schema from the source system is fine for the company web application, but isn't sufficient for our analytical users. 

Let's suppose our company wants to answer the following questions:
1. What impact does lowering a product price have on sales?
2. How much revenue did a particular product generate last month?
3. For customers that have never purchased a product, how far along in the sales funnel did they get?
4. Which cities have the most customers?
5. Which products get viewed frequently, but rarely purchased?


All of these questions can be answered with the available data, but are hard or impossible to answer with the data in the source schema. Let's make some tables that are better suited for answering these questions.

But first, let's create a database and schema to hold our tables with transformed data:
``` SQL
CREATE DATABASE analytics;

CREATE SCHEMA analytics.web_application_db;
```

#### What impact does lowering a product price have on sales?

To answer this question we'll need to find two things:
1. When did the price of a product change?, and
2. What was the volume of sales for a product before and after the price change?

The volume of sales is pretty easy to figure out from the source schema. We can write a query like this:
``` SQL
SELECT COUNT(*) as products_sold
FROM sales
WHERE sale_date between $price_change_date and dateadd(month, 1, $price_change_date);
```

But the current schema can't help us answer the question of when a products price changes. If we confirm that the price of a product will change, at most, once per day, then we can create our own price history table by taking daily snapshots of the `products` table. We'll go into more detail about how we will transform the data in Chapter 6. For now all we need to know is that we can get that data, so let's create a table to hold it.

``` SQL
CREATE TABLE analytics.web_application_db.price_history (
    id                      INT,
    product_id              INT,
    old_price               NUMERIC,
    new_price               NUMERIC,
    price_change_date       DATE,
    updated_at              TIMESTAMP_TZ
);
```

With this new table in place, we can easily this question with a query like:
``` SQL
WITH price_change as (
    SELECT product_id, max(price_change_date) as price_change_date
    FROM analytics.web_application_db.price_history
    WHERE product_id = 1234
),
sales_before as (
    SELECT COUNT(s.id) as products_sold_before
    FROM s.product_id, analytics.web_application_db.sales as s
    INNER JOIN price_change as pc
        on pc.product_id = s.product_id
    WHERE s.sale_date between pc.price_change_date and dateadd(month, -1, pc.price_change_date)
),
sales_after as (
    SELECT s.product_id, COUNT(s.id) as products_sold_after
    FROM analytics.web_application_db.sales as s
    INNER JOIN price_change as pc
        on pc.product_id = s.product_id
    WHERE s.sale_date between pc.price_change_date and dateadd(month, 1, pc.price_change_date)
)
SELECT pc.product_d, pc.price_change_date, sb.products_sold_before, sa.products_sold_after
FROM price_change as pc
INNER JOIN sales_before as sb
    on sb.product_id = pc.product_id
INNER JOIN sales_after as sa
    on sa.product_id = pc.product_id;
```
This query will show us the count of sales one month before and one month after the latest price change for product "1234". This table is structured as a "transaction" table, because it shows one row for every event (a price change). Another way we could have built this table is as a "snapshot" table, showing the price for every product on every day. A snapshot table will have a lot more records in it than a transaction table showing the same data, but it can also be more useful in answering certain types of questions. It's the Data Engineer's job to decide how to structure the table to be the most useful for their users.

In Chapter 6 we'll be populating these tables with data, and we can test out this query.

#### How much revenue did a particular product generate last month?

At first glance you might think we can answer this question using the source schema. We can find the total sales for a product last month and then multiply by the product's price. This doesn't work, however, because we only know the product's current price, not the product's price at the time it was sold. If we sold 100 products last month, and raised the price by a dollar earlier today, then we would report revenue from the product that is $100 greater than the actual revenue the product generated.

Fortunately, we just created a price history table, so let's see how we might answer this question by leveraging this new table:
``` SQL
WITH sales_with_price (
    SELECT product_id, sale_date, sale_price
    FROM (
        SELECT s.product_id,
            , s.sale_date
            , ph.new_price as sale_price
            , ROW_NUMBER() OVER (ORDER BY ph.price_change_date DESC) as rn
        FROM analytics.web_application_db.sales as s
        INNER JOIN analytics.web_application_db.price_history as ph
            on s.product_id = ph.product_id
            and ph.price_change_date < s.sale_date
        WHERE s.product_id = 1234
    )
    WHERE rn = 1
)
SELECT product_id, sum(sale_price) as total_revenue
FROM sales_with_price
WHERE sale_date > dateadd(month, -1, current_date)
GROUP BY product_id;
```

This query works, but with the window function, subquery, and CTE it's a bit complicated to both read and write. Let's see how the query changes if we use a snapshot pricing history table, instead of the transaction table we created for the last question.

``` SQL
CREATE TABLE analytics.web_application_db.pricing_snapshots (
    id                      INT,
    product_id              INT,
    current_price           NUMERIC,
    snapshot_date           DATE,
    updated_at              TIMESTAMP_TZ
);
```

With the above table in place, we can rewrite our query as:
``` SQL
WITH sales_with_price (
    SELECT s.product_id, s.sale_date, ps.current_price as sale_price
    FROM analytics.web_application_db.sales as s
    INNER JOIN analytics.web_application_db.pricing_snapshots as ps
        on ps.product_id = s.product_id
        and s.sale_date = ps.snapshot_date
    WHERE s.product_id = 1234
)
SELECT product_id, sum(sale_price) as total_revenue
FROM sales_with_price
WHERE sale_date > dateadd(month, -1, current_date)
GROUP BY product_id;
```

The snapshot table allows us to make our query easier to read and write. And as we'll see in Chapter 6, the transformation is much easier to write for a snapshot, as well. The drawback is that snapshot tables can become very large. If our e-commerce company has a million products it sells, then the snapshot table would add a million new records every day. While Snowflake is an excellent tool for handling very large tables, it also means the table will be slower to query and result in higher fees from Snowflake to process queries against that table.

For our purposes we'll use both tables, but keep these sorts of trade-offs in mind when designing your own warehouse.


#### For customers that have never purchased a product, how far along in the sales funnel did they get?

For this query we need to:
1. find the population of customers that have never purchased a product, and
2. determin "how far along in the sales funnel" they got.

Finding customers who have never purchased a product is simply done using the source schema:
``` SQL
SELECT *
FROM customers as c
WHERE c.id not in (SELECT customer_id FROM salses);
```

But we also have this phrase "how far along in the sales funnel" that doesn't obviously map to the data available. Let's assume we reached out to our stakeholders as well as the people that maintain the application DB and we found the following:
1. The `user_sessions` table is a log of every page on the website that a customer visits. 
2. When our business analysts talk about the sales funnel, they group customers into four stages: "viewed product", "added to cart", "began chackout", and "sale_complete".
3. Each of these stages correspond to a specific URL pattern stored in the `page` field of the `user_sessions` table.

Every time a new analyst wants to answer this question, they are going to have to repeat this interview process so they can the proper context to understand what is being asked and how to find it. Instead, let's create a table that is regularly updated and can easily provide the answer to this question:

``` SQL
CREATE TABLE analytics.web_application_db.customers_without_sales (
    id                      INT,
    customer_id             INT,
    last_page_visited       VARCHAR,
    last_page_visited_on    TIMESTAMP_TZ,
    max_funnel_stage        VARCHAR,
    updated_at              TIMESTAMP_TZ
);
```

We can now write a query like:
``` SQL
SELECT customer_id, max_funnel_stage
FROM analytics.web_application_db.customers_without_sales;
```

#### Which cities have the most customers?

The `customers` table in the source schema contains an `address` field that our analysts could query to get the customer city. But if our analysts are doing lots of geographic queries on our customers we can make things a lot easier for them by parsing the customer address field and persisting it in a table for them to query:

``` SQL
CREATE TABLE analytics.web_application_db.customer_details (
    id                      INT,
    customer_id             INT,
    full_name               VARCHAR,
    first_name              VARCHAR,
    last_name               VARCHAR,
    address_1               VARCHAR,
    address_2               VARCHAR,
    city                    VARCHAR,
    state                   VARCHAR,
    zip_code                VARCHAR,
    updated_at              TIMESTAMP_TZ
);
```

Parsing names and addresses is actually a tough problem to solve, and we'll talk about how we might go about it in Chapter 6.

With this table we could answer our question with:
``` SQL
SELECT city, count(*) as customer_count
FROM analytics.web_application_db.customer_details
GROUP BY city
ORDER BY count(*) DESC;
```

#### Which products get viewed frequently, but rarely purchased?

This is another question that an analyst could find the answer to with the source schema, but we could potentially persist as a table for their convenience because they query this data a lot.

``` SQL
CREATE TABLE analytics.web_application_db.product_conversion (
    id                      INT,
    product_id              INT,
    views_lifetime          INT,
    views_last_30_days      INT,
    purchases_lifetime      INT,
    purchases_last_30_days  INT,
    conversion_lifetime     NUMERIC,
    conversion_last_30_days NUMERIC,
    updated_at              TIMESTAMP_TZ
);
```

With this table an analyst could answer this question with:
``` SQL
SELECT *
FROM analytics.web_application_db.product_conversion
ORDER BY conversion_lifetime ASC
LIMIT 100;
```

## Wrapping Up
Our data warehouse now has twelve tables defined: seven that are a copy of the source system, and five that transform the data for use by our analysts.

In Chapter 4 we'll go over bringing in the data from the source system, and in Chapter 6 we'll set up systems to transform the data and put it in its new tables.

---

Next Chapter: 