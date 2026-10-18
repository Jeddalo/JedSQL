# E-Commerce Project

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
***

### 5: revenue funnel analysis
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
***
