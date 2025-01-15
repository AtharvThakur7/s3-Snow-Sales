# Retail Sales and Profitability Analysis for ElectroHub Using AWS S3 , SNOWFLAKE and POWER BI

Led a strategic sales analysis using Power BI Followed by ELT pipeline using AWS S3 for Storage and SNOWFLAKE for Transformation  to visualize key metrics like sales, profit, and quantity sold across diverse product categories. The analysis focused on identifying top-performing products, understanding sales trends, and evaluating the correlation between sales and profitability. By deriving actionable insights, the project enabled data-driven decision-making, optimizing discount strategies, and prioritizing high-performing categories to accelerate business growth and profitability.

# Project Architecture

![image alt ](https://github.com/AtharvThakur7/s3-Snow-Sales/blob/599244e6d9c136907e47adee79f2e684207707a2/Screenshot%202025-01-15%20220608.png)

## Problem Statement 

ElectroHub, a multi-category retail company, seeks to optimize its sales performance and profitability across diverse product lines and regions. To achieve this, the company requires actionable insights into key business metrics, including:

- Identifying top and bottom-performing products by sales, profit, and quantity sold.
- Analyzing sales trends over time (daily, monthly, quarterly, and annually) to understand seasonal patterns.
- Evaluating the relationship between sales and profit to identify profitability drivers.
- Comparing performance metrics (sales, profit, and quantity sold) across selected time periods.
- Understanding the impact of discount categories on sales through average discounts offered.
- Monitoring the total number of orders to gauge customer demand.
- Visualizing sales distribution across different cities.
- Identifying the top profit-generating categories to prioritize strategic focus.

## Dimension Model And Transformation 

# a> Dimension Model
![image alt ](https://github.com/AtharvThakur7/s3-Snow-Sales/blob/c2bf344f7e772ada246f8c697e2aa967bea20428/Screenshot%202025-01-15%20210607.png)


***Fact Table***: The Fact Table captures transactional data, including:

Date (dd/mm/yyyy), CustomerID, PromotionID, ProductID
Key metrics: Units Sold, Price Per Unit, Total Sales, Discount Percentage, Discount Value, and Net Sales.
This table stores the core numerical data for analysis.

***Dimension Tables*** :

- Customer: Contains customer details, providing context for sales data.
- Product: Stores product information, helping to analyze sales by category, type, etc.
- Promotion: Includes promotion details (e.g., Summer Sale, Festive Diwali, etc.), discount coupons, percentages, and values associated with each promotion.
- Date Table 1: Represents an active relationship to the Fact Table based on the transaction date, supporting time-based analysis (daily, monthly, etc.).
- Date Table 2: Represents an inactive relationship, used for specific date-based calculations such as comparing different time periods or analyzing data across alternate date ranges.

 # b>Transformation




 ***SQL Transformation in SnowFlake***:

```bash
-- lets performed tranformation and build DIMENSION MODEL

SELECT * FROM sales.raw_sales.customer;
SELECT * FROM sales.raw_sales.product;
SELECT * FROM sales.raw_sales.promotion;
SELECT * FROM sales.raw_sales.sales;

-- Tranforming the sales fact table by calculating numeric facts by joining the table and also handle null values using COALESCE()

CREATE TABLE transformed_sales AS (
    SELECT  
        s.sales_id,
        s.date_col,
        s.customer_id,
        s.promotion_id,
        s.product_id,
        s.units_sold,
        COALESCE(p.price, 0) AS price_per_unit,
        COALESCE(s.units_sold * p.price, 0) AS total_sales,
        COALESCE(pr.discount, 0) AS discount_percentage,
        (COALESCE(s.units_sold * p.price, 0) * COALESCE(pr.discount, 0)) / 100 AS discount_value,
        (COALESCE(s.units_sold * p.price, 0) - 
         (COALESCE(s.units_sold * p.price, 0) * COALESCE(pr.discount, 0)) / 100) AS net_sales,
        (COALESCE(s.units_sold * p.price, 0) * 0.1) AS profit

    FROM SALES.RAW_SALES.sales s

    LEFT JOIN SALES.RAW_SALES.product p
    ON s.product_id = p.product_id

    LEFT JOIN SALES.RAW_SALES.PROMOTION pr
    ON s.promotion_id = pr.promotion_id
);


select * from SALES.TRANFORMED_SALES.TRANSFORMED_SALES;
DROP TABLE tranformed_sales;


CREATE TABLE dim_customer AS (
    SELECT * FROM SALES.RAW_SALES.CUSTOMER
);


SELECT * FROM dim_customer;

CREATE TABLE dim_product AS (
    SELECT * FROM SALES.RAW_SALES.PRODUCT
);

SELECT * FROM dim_product;

CREATE TABLE dim_promotion AS (
    SELECT * FROM SALES.RAW_SALES.PROMOTION
);


SELECT * FROM dim_promotion;

```

***Python Transformation in SnowFlake***: 

```bash
import snowflake.snowpark as snowpark
from snowflake.snowpark.functions import col, regexp_replace, regexp_extract

def main(session: snowpark.Session): 

    # tranforming the promotion table to extract discount value from reduction_type column
    
    
    # Step 1: Load the raw promotion table from the raw_sales schema
    promotion_df = session.table('raw_sales.promotion')

    # Step 2: Replace "Buy 1 Get 1 Free" with "50" in the Price_Reduction_Type column
    promotion_df = promotion_df.with_column(
        'reduction_type',
        regexp_replace(col('reduction_type'), 'Buy 1 Get 1 Free', '50')
    )

    # Step 3: Extract numeric discount values from the 'Price_Reduction_Type' column
    promotion_df = promotion_df.with_column(
        'discount',
        regexp_extract(col('reduction_type'), r'(\d+)', 1).cast('int')
    )

    # Step 4: Overwrite the existing raw_sales.promotion table with the transformed data
    promotion_df.write.mode('overwrite').save_as_table('raw_sales.promotion')

    # Step 5: Display a sample of the transformed table in the output
    promotion_df.show()

    # Step 6: Return a preview of the updated data for the Results tab
    return promotion_df.limit(5)

```


# Business InSights
##  a> Part 1  - Sales Trends and Key Metrics Analysis

![image alt](https://github.com/AtharvThakur7/s3-Snow-Sales/blob/8d2344fbf7b0c8f90f612e93529de6af34476612/Screenshot%202025-01-15%20191003.png)


1. ***Relationship Between Sales And Profit*** :  
 The scatter plot reveals a positive correlation between sales and profit. Higher net sales generally translate to higher profits.

2. ***Sales Trends Over Time*** :

- Annual net sales show a slight decline from 2020 to 2024, peaking at 32M in 2023 but dropping in 2024.
- seasonal promotions, such as "Summer Sale" and "Weekend Flash Sale," drive significant spikes in sales during their respective periods.

 3. ***Average Discount Offered in Categories***:

- Weekend Flash Sales have the highest average discount (23K), followed by Clearance Sales (18K).

- Minimal discounts (or none) are observed for Festive Diwali and New Year Special, indicating premium pricing strategies during festive periods.

4. ***Total Orders***:

- The business processed 4K total orders, showcasing a reasonable volume for high-value products like electronics and home appliances.

5. ***Sales by City*** :

- Major cities such as Mumbai, Bangalore, Delhi, and Pune dominate sales, reflecting strong urban demand.
- Tier-2 cities, including Nagpur, Bhopal, and Visakhapatnam, contribute less, highlighting opportunities for growth and market expansion in these regions.

## b> Part 2 - Top and Bottom Product Performance

![image alt](https://github.com/AtharvThakur7/s3-Snow-Sales/blob/e4d56ba5407e53dbbacdbe5c28150f2c9300ff8f/Screenshot%202025-01-15%20191141.png)

1.***Top/Bottom 5 Products by Sales***:

- High-ticket items like Apple iPhone 14 (22M), Apple MacBook Air (21M), and Sony Bravia 55” TV (21M) dominate revenue, reflecting strong demand for premium electronics
- Products like Colgate Toothpaste (22.3K) and Dove Soap Pack (86.5K) show significantly lower sales.


2.***Top/Bottom 5 Products by Unit Sold***:

- Apple iPhone 14 (281 units) and HP Pavilion Laptop (230 units) lead in both sales and quantity sold, underscoring their popularity and trust in the market.

- Items like Nivea Body Lotion (219 units) and Tupperware Lunch Box (215 units) rank at the bottom in units sold, suggesting limited appeal or being niche products.


3.***Top/Bottom 5 Products by Profit*** :

- Apple iPhone 14 (2.25M), Apple MacBook Air (2.08M), and Sony Bravia 55” TV (2.05M) are the top profit contributors, driven by their premium pricing and strong demand.

- Products like Tupperware Lunch Box (28K) and Colgate Toothpaste (2K) yield minimal profits, likely due to low margins or small market sizes


## c> Part 3  - Comparative Sales and Profit Analysis

![image alt](https://github.com/AtharvThakur7/SalesData-Analytics/blob/ea153fde828db80cde7fbdc529fcd90357d4a167/Screenshot%202025-01-07%20222413.png)


1. ***Comparison of Metrics Between Periods*** :

- The dashboard facilitates direct comparison of sales, profit, and units sold for two custom timeframes.

 - From the given trends, consistent sales (~122M) and profits (~12.2M) suggest a steady performance.



## Conclusion and Impact

1. ***Conclusion*** :

- The electronics category continues to lead in sales and profitability, proving its central role in the business strategy.

- While flash sales drive volume, discounts must be optimized to sustain profitability.
  
- Focus on high-value cities and improve marketing for underperforming product categories.



2. ***Business Impact***:

- Improving profitability requires targeting high-margin products and optimizing promotional expenses.

- Geographic expansion in Tier 2 cities and festive sales can sustain long-term growth.

- Regularly updating the product portfolio ensures competitiveness and mitigates the risk of declining trends.

  
  


  


