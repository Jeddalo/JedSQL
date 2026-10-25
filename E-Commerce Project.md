# E-Commerce Sales Funnel Analysis

### 📌 Business Tasks
E-commerce business wants to evaluate website and data analytics team has been tasked to evaluate business website performance and provide actionable business recommendations.

### Business Deliverables
The key areas covered for analysis include:
- **Sales Funnel Stages:** Defining and counting user progression through five key stages: *Page View*, *Add to Cart*, *Checkout Start*, *Payment Info*, and *Purchase*
- **Conversion Rate Optimization:** Calculating conversion rates between funnel stages to identify bottlenecks.
- **Marketing Channel Effectiveness:** Comparing traffic sources (*Organic*, *Paid Ads*, *Email*, *Social Media*) to determine which channels drive the most valuable customer behavior.
- **Customer Journey Timing:** Analyzing the average time taken for users to move through different stages of the funnel.
- **Revenue Metrics:** Evaluating financial KPIs such as *Average Order Value (AOV)* and comparing them against *Customer Acquisition Cost (CAC)*

## 📌 Data Exploration

### 1: Define sales funnel and different stages

````sql
WITH funnel_stages AS (
	SELECT
		COUNT(DISTINCT CASE WHEN event_type = 'page_view' THEN user_id END) AS stage_1_views,
		COUNT(DISTINCT CASE WHEN event_type = 'add_to_cart' THEN user_id END) AS stage_2_cart,
		COUNT(DISTINCT CASE WHEN event_type = 'checkout_start' THEN user_id END) AS stage_3_checkout,
		COUNT(DISTINCT CASE WHEN event_type = 'payment_info' THEN user_id END) AS stage_4_payment,
		COUNT(DISTINCT CASE WHEN event_type = 'purchase' THEN user_id END) AS stage_5_purchase	
	FROM user_events
	WHERE event_date >= DATEADD(DAY, -30, (SELECT MAX(event_date) FROM user_events))
)
````
**Answer:**

<img width="522" height="79" alt="Image" src="https://github.com/user-attachments/assets/5933effa-2366-4cd8-9873-bda6a6adc6bd" />

***

### 2: Conversion rates through the funnel
````sql
WITH funnel_stages AS (
	SELECT
		COUNT(DISTINCT CASE WHEN event_type = 'page_view' THEN user_id END) AS stage_1_views,
		COUNT(DISTINCT CASE WHEN event_type = 'add_to_cart' THEN user_id END) AS stage_2_cart,
		COUNT(DISTINCT CASE WHEN event_type = 'checkout_start' THEN user_id END) AS stage_3_checkout,
		COUNT(DISTINCT CASE WHEN event_type = 'payment_info' THEN user_id END) AS stage_4_payment,
		COUNT(DISTINCT CASE WHEN event_type = 'purchase' THEN user_id END) AS stage_5_purchase	
	FROM user_events
	WHERE event_date >= DATEADD(DAY, -30, (SELECT MAX(event_date) FROM user_events)) -- 30 days from max date on date column
	 
)
	SELECT 
		stage_1_views,
		stage_2_cart,
		CAST(stage_2_cart * 100 / stage_1_views  as decimal(5,2)) AS view_to_cart_rate,

		stage_3_checkout,
		CAST(stage_3_checkout * 100 / stage_2_cart  as decimal(5,2)) AS cart_to_checkout_rate,

		stage_4_payment,
		CAST(stage_4_payment * 100 / stage_3_checkout  as decimal(5,2)) AS checkout_to_payment_rate,

		stage_5_purchase,
		CAST(stage_5_purchase * 100 / stage_4_payment  as decimal(5,2)) AS payment_to_purchase_rate,

		CAST(stage_4_payment * 100 / stage_1_views  as decimal(5,2)) AS checkout_to_payment_rate

	FROM funnel_stages;
````
**Answer:**

<img width="1093" height="77" alt="Image" src="https://github.com/user-attachments/assets/6b64630f-b5ee-438f-a4dd-1b1a4de743c8" />

***

### 3: Funnel by Source
````sql
WITH source_funnel AS (
	SELECT
		traffic_source,
		COUNT(DISTINCT CASE WHEN event_type = 'page_view' THEN user_id END) AS views,
		COUNT(DISTINCT CASE WHEN event_type = 'add_to_cart' THEN user_id END) AS carts,
		COUNT(DISTINCT CASE WHEN event_type = 'purchase' THEN user_id END) AS purchases
	FROM user_events
	WHERE event_date >= DATEADD(DAY, -30, (SELECT MAX(event_date) FROM user_events))
	GROUP BY traffic_source
)

	SELECT
		traffic_source,
		views,
		carts,
		purchases,
		CAST(carts * 100 / views  as decimal(5,2)) AS cart_conversion_rate,
		CAST(purchases * 100 / views  as decimal(5,2)) AS cart_conversion_rate,
		CAST(purchases * 100 / carts as decimal(5,2)) AS cart_to_purchase_conversion_rate
	FROM source_funnel
	ORDER BY purchases;
````
**Answer:**

<img width="675" height="92" alt="Image" src="https://github.com/user-attachments/assets/119511dc-bae3-451f-87dc-a2e154704ce7" />

***

### 4: Time to conversion analysis
````sql
WITH user_journey AS (
	SELECT
		user_id,
		MIN(CASE WHEN event_type = 'page_view' THEN event_date END) AS view_time,
		MIN(CASE WHEN event_type = 'add_to_cart' THEN event_date END) AS cart_time,
		MIN(CASE WHEN event_type = 'purchase' THEN event_date END) AS purchase_time
	
	FROM user_events
	WHERE event_date >= DATEADD(DAY, -30, (SELECT MAX(event_date) FROM user_events))
	GROUP BY user_id
	HAVING MIN(CASE WHEN event_type = 'purchase' THEN event_date END) IS NOT NULL
)

	SELECT
		COUNT(*) AS converted_users, 
		CAST(AVG(DATEDIFF(MINUTE, view_time, cart_time)) as decimal(5,2)) AS avg_view_to_cart_minutes,
		CAST(AVG(DATEDIFF(MINUTE, cart_time, purchase_time)) as decimal(5,2)) AS avg_cart_to_purchase_minutes,
		CAST(AVG(DATEDIFF(MINUTE, view_time, purchase_time )) as decimal(5,2)) AS avg_total_to_journey_minutes
	FROM user_journey;
````
**Answer:**

<img width="600" height="54" alt="Image" src="https://github.com/user-attachments/assets/634f223e-dd26-4359-8b21-5fc7902c56ea" />

***

### 5: Revenue funnel analysis
````sql
WITH funnel_revenue AS (
	SELECT
		COUNT(DISTINCT CASE WHEN event_type = 'page_view' THEN user_id END) AS total_visitors,
		COUNT(DISTINCT CASE WHEN event_type = 'purchase' THEN user_id END) AS total_buyers,
	    CAST(SUM(CASE WHEN event_type = 'purchase' THEN amount END)as decimal(18,2)) AS total_revenue,
		COUNT(CASE WHEN event_type = 'purchase' THEN 1 END) AS total_orders
	FROM user_events

	WHERE event_date >= DATEADD(DAY, -30, (SELECT MAX(event_date) FROM user_events))
)
	SELECT 
	total_visitors,
	total_buyers,
	total_orders,
	total_revenue,
	CAST(total_revenue / total_orders as decimal(5,2)) AS avg_order_value,
	CAST(total_revenue / total_buyers as decimal(5,2)) AS revenue_per_buyer,
	CAST(total_revenue / total_visitors as decimal(5,2)) AS revenue_per_visitor
	FROM funnel_revenue;
````
**Answer:**

<img width="645" height="42" alt="Image" src="https://github.com/user-attachments/assets/1fb3f266-f20b-4a85-a17a-d425d0d13bdb" />

***

## 📌Insights and Business Recommendation

1. **Funnel Source Conversion:** Analysis provided a detailed breakdown of conversion rates across sales funnel. The calculated results are as follows:
```
   		- ** View to Cart:** 30%
		- ** Cart to Checkout:** 71%
		- ** Checkout to Payment:** 81%
		- ** Payment to Checkout:** 92%
```
Following this analysis, the folloeing is recommended:
- **Optimize User Experience** The low **view-to-cart rate** suggests that while traffic exists, Users may not be finding products efficiently, I recommend an investigation into website navigation and product display.
- Validate Technical Performance: The high **payment-to-purchase conversion (92%)** indicates that the technical infrastructure for final transactions is functioning without significant friction or bottleneck.

2. **Marketing Channel Effectiveness:** Analysis provided shows how the different traffic sources - *Organic*, *Paid Ads, *Email*, and *Social Media* perform in driving conversions. While **Social Media** drives a high volume of page views, it shows significantly lower conversion rates compared to other channels. **Email marketing**,however, demonstrates a mush higher efficiency, with notable conversion rate of 63% from view to cart.

Following this analysis, the following is recommended:
- **Address Vanity Metrics:** High traffic from Social Media can be misleading if it does not translate into actual purchase.
- **Reallocate Budget:** The business should consider reducing investment in underperforming Social Media channels.
- **Double Down on High- Performing Channels:** Resources should be shifted toward Email marketing, which proves to be more effective at turning visitors.

3. **Customer Journey Timing:** Analysis provided the measures of the time spend transitioning between specific stages of the sales funnel. The findings for users who completed a purchase are as follows:
   
```
View to Cart: Average of 11 minutes.
Cart to Purchase: Average of 13 minutes.
Total Journey: Average of 24 minutes.
```
The durations appear realistic and do not show extreme values that might suggest bot activity.

Following this analysis, the following is recommended:

**Process Efficiency:** The business should consider investigating whether these average times align with their internal expectations for the user experience.

**Business Inquiry:** Use this data to opena asialogue with stakeholders regarding whether the *24-minute average journey* can be optimized to make the path to purchase faster and more efficient.

