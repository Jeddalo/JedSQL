# 👩🏻‍⚕️ Medical-Appointment-No-Show Analysis

### 📌 Business Tasks
Using a dataset of over 100,000 medical appointment to analyze factors that influence whether patients fail to show up for their scheduled visits.

## 📌 Data Exploration

### 1: What's our overall no-show rate?

````sql
SELECT 
	No_show,
	COUNT(*) as total_appointments,
	CAST(COUNT(*) *100.0/ (SELECT COUNT(*) FROM medicalappointment) AS decimal(5,2)) AS
pct_of_total
FROM medicalappointment
GROUP BY No_show;
````
**Answer:**

<img width="303" height="101" alt="Image" src="https://github.com/user-attachments/assets/d7fe5d2d-bbea-4a80-8437-fad6a4b20c06" />

***

### 2: Does the day of the week matter

````sql
SELECT 
	DATENAME(WEEKDAY,AppointmentDay) as appointment_day,
	COUNT(*) AS total_appointments,
	SUM(CASE WHEN No_show = 'Yes' THEN 1 ELSE 0 END) as no_shows, -- clean  way to count specific condition without subquery
	CAST(SUM(CASE WHEN No_show = 'Yes' THEN 1 ELSE 0 END) * 100.0 / COUNT(*) as decimal(5,2)) as rate_of_no_shows
FROM medicalappointment
GROUP BY DATENAME(WEEKDAY,AppointmentDay)
ORDER BY rate_of_no_shows DESC;
````
**Answer:**

<img width="423" height="172" alt="Image" src="https://github.com/user-attachments/assets/23da52c3-1872-4521-bfb0-2a65787e50b7" />

***

### 3: Does lead time matter

````sql
SELECT 
	CASE
		WHEN lead_time_days = 0 THEn 'Same Day'
		WHEN lead_time_days BETWEEN 1 AND 3 THEN 'Short (1-3 Days)'
		WHEN lead_time_days BETWEEN 4 AND 7 THEN 'Within a week'
		ELSE 'Long Lead (8+ days)'
	END as lead_time_bucket,
	COUNT(*) as total_appointments,
	CAST(SUM(CASE WHEN No_show = 'Yes' THEN 1 ELSE 0 END) * 100.0  / COUNT(*) as decimal(5,2)) as no_show_rate
FROM medicalappointment
GROUP BY 
	CASE
		WHEN lead_time_days = 0 THEn 'Same Day'
		WHEN lead_time_days BETWEEN 1 AND 3 THEN 'Short (1-3 Days)'
		WHEN lead_time_days BETWEEN 4 AND 7 THEN 'Within a week'
		ELSE 'Long Lead (8+ days)' 
	END
ORDER BY no_show_rate DESC;
````
**Answer:**

<img width="346" height="106" alt="Image" src="https://github.com/user-attachments/assets/3255fc99-d235-45f1-ab8e-ff9d0f8c4ef0" />

***

### 4: Does Age affect no-show rates


````sql
With Age_group_bucket AS ( --Using CTE this time
	SELECT 
		CASE
			WHEN Age BETWEEN 0 and 12 THEN 'child'
			WHEN Age BETWEEN 13 and 19 THEN 'Teen'
			WHEN Age BETWEEN 20 and 39 THEN 'Young Adult'
			WHEN Age BETWEEN 40 and 59 THEN 'Adult'
			ELSE 'Senior'
		END AS age_group,
		No_show
	FROM medicalappointment
)
SELECT
	age_group,
	COUNT(*) as total_appointments,
	CAST(SUM(CASE WHEN No_show = 'Yes' THEN 1 ELSE 0 END) * 100.0  / COUNT(*) as decimal(5,2)) as no_show_rate
	FROM Age_group_bucket
	GROUP BY age_group
	ORDER BY no_show_rate DESC;
````
**Answer:**

<img width="289" height="121" alt="Image" src="https://github.com/user-attachments/assets/238abdfa-505c-48d5-a059-75c754f68b8a" />

***

### 5: Does SMS reminders help

````sql
SELECT 
	CASE WHEN SMS_received = 1 THEN 'Received SMS' ELSE 'No SMS'
	END AS sms_status,
	COUNT(*) as total_appointments,
	CAST(SUM(CASE WHEN No_show = 'Yes' THEN 1 ELSE 0 END) * 100.0  / COUNT(*) as decimal(5,2)) as no_show_rate
	FROM medicalappointment
	GROUP BY CASE WHEN SMS_received = 1 THEN 'Received SMS' ELSE 'No SMS'
	END;
````
**Answer:**

<img width="324" height="65" alt="Image" src="https://github.com/user-attachments/assets/2b543ee8-f704-4c52-971f-42ed55066b80" />

***

### 6: Which neighbourhoods have the highest risk?

````sql
SELECT TOP 15
	Neighbourhood,
	COUNT(*) as total_appointments,
	CAST(SUM(CASE WHEN No_show = 'Yes' THEN 1 ELSE 0 END) * 100.0  / COUNT(*) as decimal(5,2)) as no_show_rate,
	RANK() OVER (ORDER BY  SUM(CASE WHEN No_show = 'Yes' THEN 1 ELSE 0 END) * 100.0 / COUNT(*) DESC) as risk_rank
FROM medicalappointment
GROUP BY Neighbourhood
HAVING COUNT(*) >= 100
ORDER BY no_show_rate DESC;
````
**Answer:**

<img width="428" height="311" alt="Image" src="https://github.com/user-attachments/assets/46b25717-760d-4b0f-9e21-1736cebc847f" />

***

### 7: Patient-level risk scoring

````sql
SELECT
	patientid,
	AppointmentID,
	AppointmentDay,
	No_show,
	Count(*) OVER (
		PARTITION BY PatientID
		ORDER BY AppointmentDay
		ROWS BETWEEN UNBOUNDED PRECEDING AND 1 PRECEDING) as prior_appointments,
	SUM(CASE WHEN No_show = 'Yes' THEN 1 ELSE 0 END) OVER 
		(PARTITION BY PatientID
		ORDER BY AppointmentDay
		ROWS BETWEEN UNBOUNDED PRECEDING AND 1 PRECEDING) as prior_no_shows
FROM medicalappointment
ORDER BY PatientId, AppointmentDay;
````
**Answer:**

<img width="537" height="213" alt="Image" src="https://github.com/user-attachments/assets/63801e44-8bfc-4369-a791-85f0fd62afc3" />
