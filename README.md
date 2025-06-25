# Introduction
This project explores customer retention, lifetime value, and retention of an e-commerce company with the goal of improving retention to maximise revenue.

[Click here](https://www.lukebarousse.com/int-sql) for the data!

### Business Questions
1. Customer Segmentation: Who are our most valuable customers?
2. Cohort Analysis: How do different customer groups generate revenue?
3. Retention Analysis: Which customers haven't purchased recently?

### Tools I used
- **SQL**: The backbone for my analysis, allowing me to query the database and uncover insights
- **DBeaver**: Code editor. 
- **Git & Github**: Version control and sharing, ensuring collaboration and tracking. 

# The Analysis

### Creating View
Since analysis will be using cohort data, here is the script! 

*Use "DROP VIEW cohort_analysis;" as the first line if you would like to alter the view.*

```sql
CREATE OR REPLACE VIEW cohort_analysis
AS WITH customer_revenue AS (
         SELECT s.customerkey,
            s.orderdate,
            sum(s.quantity::double precision * s.netprice * s.exchangerate) AS total_net_revenue,
            count(s.orderkey) AS num_orders,
            c.countryfull,
            c.age,
            c.givenname,
            c.surname
           FROM sales s
             LEFT JOIN customer c ON c.customerkey = s.customerkey
          GROUP BY s.customerkey, s.orderdate, c.countryfull, c.age, c.givenname, c.surname
        )
 SELECT customerkey,
    orderdate,
    total_net_revenue,
    num_orders,
    countryfull,
    age,
    CONCAT(TRIM(givenname), ' ', TRIM(surname)) AS cleaned_name,
    min(orderdate) OVER (PARTITION BY customerkey) AS first_purchase_date,
    EXTRACT(year FROM min(orderdate) OVER (PARTITION BY customerkey)) AS cohort_year
FROM customer_revenue AS cr;
```

### 1. Customer Segmentation
- Categorised customers based on total lifetime value (LTV)
- Assigned customers to High, Mid, and Low-value segments
- Calculated total revenue

```sql
WITH customer_ltv AS (
	SELECT 
		customerkey,
		cleaned_name,
		sum(total_net_revenue) AS total_ltv
	FROM cohort_analysis 
	GROUP BY 
		customerkey,
		cleaned_name
), customer_segments AS (
	SELECT 
		PERCENTILE_CONT(0.25) WITHIN GROUP (ORDER BY total_ltv) AS ltv_25th_percentile,
		PERCENTILE_CONT(0.75) WITHIN GROUP (ORDER BY total_ltv) AS ltv_75th_percentile
	FROM customer_ltv
) , segment_values AS (
	SELECT 
		c.*,
		CASE 
				WHEN c.total_ltv < cs.ltv_25th_percentile THEN '1 - Low-Value'
				WHEN c.total_ltv <= cs.ltv_75th_percentile THEN '2- Mid-Value'
				ELSE '3 - High-Value'
		END AS customer_segment
	FROM 
		customer_ltv AS c, 
		customer_segments AS cs 
)

SELECT 
	customer_segment,
	SUM(total_ltv) AS total_ltv,
	COUNT(customerkey) AS customer_count,
	SUM(total_ltv) / COUNT(customerkey) AS avg_ltv
FROM segment_values
GROUP BY 
	customer_segment
ORDER BY
	customer_segment DESC
```
| Customer Segment| Total LTV| Customer Count| Avg LTV|
|-----------|-------------|-------------|-------------|
|3 - High-Value|135,429,277|12,372|10,946|
|2- Mid-Value|66,636,451|24,743|2,693|
|1 - Low-Value|4,341,809|12,372|350|

Key findings: 
- **High-value segment** (25% of customers) drives 66% of revenue ($135.4M), hence losing 1 significantly impacts revenue.
- **Mid-value segment** (50% of customers) generates 32% of revenue ($66.6M). Personalised promotions may encourage more purchases.
- **Low-value segment** (25% of customers) accounts for 2% of revenue ($4.3M). Engagement campaigns and relevant pricing strategies could be implemented to increase purchase frequency. 

### 2. Cohort Analysis
- Tracked revenue and customer count per cohort
- Cohorts were grouped by year of first purchase
- Analysed customer retention at a cohort level

```sql
SELECT 
	cohort_year,
	count(DISTINCT customerkey) AS total_customers,
	sum(total_net_revenue) AS total_revenue,
	sum(total_net_revenue) / count(DISTINCT customerkey) AS customer_revenue
FROM cohort_analysis
WHERE orderdate = first_purchase_date 
GROUP BY 
	cohort_year
```

| Cohort Year| Total Customers| Total Revenue| Customer Revenue|
|-----------|-------------|-------------|-------------|
|2015|2,825|7,245,612|2,564|
|2016|3,397|9,839,134|2,896|
|2017|4,068|11,771,496|2,893|
|2018|7,446|19,773,770|2,655|
|2019|7,755|22,245,058|2,868|
|2020|3,031|7,058,614|2,328|
|2021|4,663|11,974,082|2,567|
|2022|9,010|21,507,554|2,387|
|2023|5,890|12,890,580|2,188|
|2024|1,402|2,764,779|1,972|

Key findings:
- Revenue per customer shows a decreasing trend from 2022
- 2023 and 2024 show a significant drop in total customers 

### 3. Customer Retention
- Identified customers at risk of churning
- Analysed last purchase patterns
- Calculated customer-specific metrics

```sql
WITH customer_last_purchase AS (
	SELECT 
		customerkey,
		cleaned_name,
		orderdate,
		ROW_NUMBER () OVER (PARTITION BY customerkey ORDER BY orderdate DESC) AS rn, 
		first_purchase_date,
		cohort_year
	FROM 
		cohort_analysis 
), churned_customers AS (

	SELECT 
		customerkey,
		cleaned_name, 
		orderdate AS last_purchased_date,
		CASE 
			WHEN orderdate < (SELECT MAX(orderdate) FROM sales) - INTERVAL '6 months' THEN 'Churned'
			ELSE 'Active'
		END AS customer_status,
		cohort_year 
	FROM customer_last_purchase 
	WHERE rn = 1 
		AND first_purchase_date < (SELECT MAX(orderdate) FROM sales) - INTERVAL '6 months'
)

SELECT 
	cohort_year,
	customer_status,
	COUNT(customerkey) AS num_customers,
	SUM(COUNT(customerkey)) OVER (PARTITION BY cohort_year) AS total_customers,
	ROUND(COUNT(customerkey) / SUM(COUNT(customerkey)) OVER (PARTITION BY cohort_year), 2) AS status_percentage
FROM churned_customers 
GROUP BY cohort_year, customer_status 
```

|Cohort Year|Customer Status|Num Customers|Total Customers|Status Percentage|
|-----------|---------------|-------------|---------------|-----------------|
|2015|Active|237|2,825|0.08|
|2015|Churned|2,588|2,825|0.92|
|2016|Active|311|3,397|0.09|
|2016|Churned|3,086|3,397|0.91|
|2017|Active|385|4,068|0.09|
|2017|Churned|3,683|4,068|0.91|
|2018|Active|704|7,446|0.09|
|2018|Churned|6,742|7,446|0.91|
|2019|Active|687|7,755|0.09|
|2019|Churned|7,068|7,755|0.91|
|2020|Active|283|3,031|0.09|
|2020|Churned|2,748|3,031|0.91|
|2021|Active|442|4,663|0.09|
|2021|Churned|4,221|4,663|0.91|
|2022|Active|937|9,010|0.10|
|2022|Churned|8,073|9,010|0.90|
|2023|Active|455|4,718|0.10|
|2023|Churned|4,263|4,718|0.90|

Key findings:
- High churn rates of >90% over the years suggests long term retention pattern.
- Retention rates are consistently low (8-10%) across all cohorts, suggesting retention issues.

# Recommendations
**1. Customer Optimisation**: Design promotions to target different segments (1 for high value, 1 for mid value, and 1 for low value customers).

**2. Cohort Performance**: Target 2022 - 2024 cohorts with re-engagement offers to encourage spending, increasing revenue per customer.

**3. Churn Prevention**: Implement a proactive intervention system to discourage churn (e.g. personalised promotion when the customer has yet to re-purchase within 6 months). 

# Conclusion
Being my second project on SQL, this project demonstrates SQL at an intermediate level by applying functions like window functions, CTEs, views, case statements, and over to sales data, providing actionable insights for the company. 
