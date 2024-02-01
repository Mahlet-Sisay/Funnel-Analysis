# Funnel-Analysis

## Table of contents
- [Project Overview](#project-overview)
- [Tools](#tools)
- [Data Sources](#data-sources)
- [Database Schema](#database-schema)
- [Exploratory Data Analysis](#exploratory-data-analysis)
- [Data Analysis](#data-analysis)
- [Tableau Visualization Results](#tableau-visualization-results)
- [Result and Findings](#result-and-findings)
- [Recommendation](#recommendation)
- [Project Presentation](#project-presentation)
  
### Project Overview

This data analysis project aims to analyze the customer funnel of Metrocar, a ride - sharing app (similar to Uber/Lift), to understand key drop-off points at various stages of the user journey and to identify areas for improvment and optimization. 

### Tools

- postgres SQL - to query the data and for Analysis   
- Tableau - to create a dinamic funnel dashboard [click here](https://public.tableau.com/authoring/metrocarsfunnelsummary/Metrocarsfunnelsummary)

### Data Sources
postgres://Test:bQNxVzJL4g6u@ep-noisy-flower-846766-pooler.us-east-2.aws.neon.tech/Metrocar

### Database Schema

![Picture1](https://github.com/Mahlet-Sisay/Funnel-Analysis/assets/137247807/5fa1e524-48d3-494b-99f2-1db6e9a1036c)
Note: dbdiagram is used to visualize this database schema

### Exploratory Data Analysis 
The funnel analysis and recommendations were made based on the following 5 key business questions:
1. What steps of the funnel should we research and improve? Are there any specific drop-off points preventing users from completing their first ride? 
2. Metrocar currently supports 3 different platforms: ios, android, and web. To recommend where to focus our marketing budget for the upcoming year, what insights can we make based on the platform?
3. What age groups perform best at each stage of our funnel? Which age group(s) likely contain our target customers?
4. Surge pricing is the practice of increasing the price of goods or services when there is the greatest demand for them. If we want to adopt a price-surging strategy, what does the distribution of ride requests look like throughout the day?
5. What part of our funnel has the lowest conversion rate? What can we do to improve this part of the funnel?

### Data Analysis 
SQL query results
1. Funnel analysis using user level granularity
```sql
WITH users AS (
  SELECT
  	1 funnel_step, 'App download' funnel_name, COUNT(*) user_count
  FROM app_downloads 
UNION ALL
  SELECT
  	2 funnel_step, 'user signup' funnel_name, COUNT(*) user_count
  FROM signups
UNION ALL
  SELECT 
  	3 funnel_step, 'Ride requested' funnel_name, COUNT(DISTINCT user_id) user_count
  FROM ride_requests
UNION ALL
  SELECT
    4 funnel_step, 'Ride accepted' funnel_name, COUNT(DISTINCT user_id) user_count
FROM ride_requests WHERE accept_ts IS NOT NULL
UNION ALL
  SELECT 
  	5 funnel_step, 'Ride completed' funnel_name, COUNT(DISTINCT user_id) user_count
FROM ride_requests
WHERE dropoff_ts IS NOT NULL 
UNION ALL
  SELECT
		6 funnel_step, 'Payment' funnel_name, COUNT(DISTINCT user_id) user_count
FROM ride_requests
JOIN transactions USING (ride_id)
WHERE transactions.charge_status ='Approved'
UNION ALL
  SELECT
    7 funnel_step, 'Reviews' as funnel_name, COUNT (DISTINCT user_id) user_count
FROM reviews )
SELECT funnel_step, funnel_name, user_count,
       --lag(user_count,1)over(order by funnel_step) as previous_num,
       ROUND((user_count::numeric/lag(user_count, 1) OVER (ORDER BY funnel_step))*100,2) percent_of_previous,
       ROUND(user_count::NUMERIC / FIRST_VALUE(user_count) OVER (ORDER BY funnel_step)*100,2) AS percent_of_top
FROM users ORDER BY 1;
```
2. Funnel analysis using ride level granularity
```sql
WITH users AS (
  /*SELECT
  	1 funnel_step, 'App download' funnel_name, NULL::INTEGER ride_count
  FROM app_downloads 
UNION distinct
  SELECT
  	2 funnel_step, 'user signup' funnel_name, NULL::INTEGER ride_count
  FROM signups
UNION distinct*/
  SELECT 
  	3 funnel_step, 'Ride requested' funnel_name, COUNT(ride_id) ride_count
  FROM ride_requests
UNION ALL
  SELECT
    4 funnel_step, 'Ride accepted' funnel_name, COUNT(ride_id) ride_count
FROM ride_requests WHERE accept_ts IS NOT NULL
UNION ALL
  SELECT 
  	5 funnel_step, 'Ride completed' funnel_name, COUNT(ride_id) ride_count
FROM ride_requests
WHERE dropoff_ts IS NOT NULL 
UNION ALL
  SELECT
		6 funnel_step, 'Payment' funnel_name, COUNT(ride_id) ride_count
FROM ride_requests
INNER JOIN transactions USING (ride_id)
WHERE transactions.charge_status ='Approved'
UNION ALL
  SELECT
    7 funnel_step, 'Reviews' as funnel_name, COUNT (ride_id) ride_count
FROM reviews )
SELECT *,
       --funnel_step, funnel_name, ride_count,
       --lag(ride_count,1)over(order by funnel_step) as previous_num,
       ROUND((ride_count::numeric/lag(ride_count, 1) OVER (ORDER BY funnel_step))*100,2) percent_of_previous,
       ROUND(ride_count::NUMERIC / FIRST_VALUE(ride_count) OVER (ORDER BY funnel_step)*100,2) AS percent_of_top
FROM users ORDER BY 1;
```
3. Query for platform insight
```sql
SELECT
  platform,
  COUNT(*) number_of_requests
FROM
  ride_requests t3
  LEFT JOIN signups t4 USING (user_id)
  LEFT JOIN app_downloads t1 ON t1.app_download_key = t4.session_id
GROUP BY
  1
ORDER BY
  2 DESC
```
4. Query for hourly distribution of ride requests
```sql
SELECT
  count(ride_id) as number_of_rides,
  extract(HOUR from request_ts) as hours,
  CASE
    WHEN EXTRACT(HOUR FROM request_ts) = 0 THEN '12 AM'
    WHEN EXTRACT(HOUR FROM request_ts) < 12 THEN CONCAT(
         EXTRACT(HOUR FROM request_ts), ' AM')
    WHEN EXTRACT(HOUR FROM request_ts) = 12 THEN '12 PM'
    ELSE CONCAT(EXTRACT(HOUR FROM request_ts) - 12,' PM')
  END AS pick_hour_12_hr
FROM
  ride_requests
WHERE
  request_ts IS NOT null
GROUP BY
  pick_hour_12_hr,
  hours
ORDER BY
  hours;
```
5. SQL query for funnel summary
```sql
WITH users AS (
  SELECT
  	app_download_key,
  	user_id,
  	platform,
  age_range,
date (download_ts) as download_dt
  FROM app_downloads
  LEFT JOIN signups
  on app_downloads.app_download_key = signups.session_id),
  downloads as
  (SELECT 0 as step,
    'downloads' as name,
    platform,
    age_range,
    download_dt,
    COUNT(DISTINCT app_download_key) as users_count,
    0 as ride_counts
    FROM users
    GROUP BY platform, age_range, download_dt),
    signup as 
    (SELECT 1 as step,
      'signups' as name,
      users.platform,
      users.age_range,
      users.download_dt,
      COUNT(DISTINCT user_id) as users_count,
      0 as ride_counts
      FROM signups
      JOIN users USING (user_id)
      WHERE signup_ts IS NOT NULL
      GROUP BY users.platform, users.age_range, users.download_dt),
      requested as 
      (SELECT 2 as step,
        'ride_requested' as name,
      users.platform,
      users.age_range,
      users.download_dt,
      COUNT(DISTINCT user_id) as users_count,
      COUNT(DISTINCT ride_id) as ride_counts
      FROM ride_requests
      JOIN users using (user_id)
      WHERE request_ts IS NOT NULL
      GROUP BY users.platform, users.age_range, users.download_dt),
        accepted as
        (SELECT 3 as step,
          'ride_accepted' as name,
      users.platform,
      users.age_range,
      users.download_dt,
      COUNT(DISTINCT user_id) as users_count,
     COUNT(DISTINCT ride_id) as ride_counts
      FROM ride_requests
      JOIN users using (user_id)
      WHERE accept_ts IS NOT NULL
      GROUP BY users.platform, users.age_range, users.download_dt),
        completed as
        (SELECT 4 as step,
          'ride_completed' as name,
      users.platform,
      users.age_range,
      users.download_dt,
      COUNT(DISTINCT user_id) as users_count,
      COUNT(DISTINCT ride_id) as ride_counts
      FROM ride_requests
      JOIN users using (user_id)
      WHERE cancel_ts IS NULL
      GROUP BY users.platform, users.age_range, users.download_dt),
          payment as
        (select 5 as step,
          'payment' as name,
      users.platform,
      users.age_range,
      users.download_dt,
     COUNT(DISTINCT user_id) as users_count,
     COUNT(DISTINCT ride_id) as ride_counts
     FROM transactions
     JOIN ride_requests using (ride_id)    
     JOIN users using (user_id)
     WHERE charge_status = 'Approved'
     GROUP BY users.platform, users.age_range, users.download_dt),
          review as
        (SELECT 6 as step,
          'review' as name,
      users.platform,
      users.age_range,
      users.download_dt,
      COUNT(DISTINCT user_id) as users_count,
      COUNT(DISTINCT ride_id) as ride_counts
      FROM reviews    
      JOIN users using (user_id)
      GROUP BY users.platform, users.age_range, users.download_dt
        )
SELECT step, name, platform, age_range, download_dt,
      users_count,
      ride_counts
      FROM (
  SELECT * FROM downloads
  UNION ALL
  SELECT * FROM signup
  UNION ALL
  SELECT * FROM requested
  UNION ALL
  SELECT * FROM accepted
  UNION ALL
  SELECT * FROM completed
  UNION ALL
  SELECT * FROM payment
  UNION ALL
  SELECT * FROM review
) AS cd
ORDER BY step,name, platform, age_range, download_dt,
      users_count,
      ride_counts;
```
### Tableau Visualization Results
1. User level funnel
- [Download here](https://public.tableau.com/authoring/metrocar_userlevelgranularity/Sheet1#1)
2. Ride level funnel
- [Download here](https://public.tableau.com/authoring/Metrocarfunnel_ridelevelgranularity/Sheet12#1)
3. Platform
- [Download here](https://public.tableau.com/authoring/metrocarsfunnelsummary/Agegroupbasedinsight2/Platform%20based%20insight#1)
4. Age range
- [Download here](
5. Hourly distribution
- [Download here](https://public.tableau.com/authoring/metrocarsfunnelsummary/Hourlyriderequestdistribution#1)

### Result and Findings
User Funnel Drop-off Points:
- Ride Accepted to Ride Completed: Approximately 50% of users who had their rides accepted did not complete those rides. 
- Signup to Ride Requested: Among the 17,623 users who completed the sign-up process, only roughly 70.40% of them proceeded to request a ride. Further analysis, such as A/B testing, is recommended to understand and improve this transition.
- Download to Signup: There is an approximate 25% decline in the number of users who download the app compared to those who complete the sign-up process.

Ride Funnel Drop-off Point:

- Ride Requested to Ride Accepted: Out of a total of 385,477 ride requests, only around 65% of these requests were accepted by drivers.

Additional Insights:

- Platform Usage: IOS users generated majority of ride requests, constituting approximately 60% of all requests.
- Target Audience: The primary customer base for Metrocar falls within the age range of 35 to 44, followed by the 25-34 age group.
- Peak Hour Strategy: Detailed analysis of ride request distribution throughout the day highlights high demand during specific time slots, such as mornings (8AM - 10AM), afternoons (4PM - 6PM), and early evenings (6PM - 8PM). This suggests the potential effectiveness of implementing a price surging strategy during peak hours.

### Recommendation
Metrocar's funnel analysis offers valuable insights into user behavior and ride dynamics, emphasizing critical areas for improvement and optimization. Addressing these drop-off points and catering to peak-hour demand could enhance service efficiency and customer satisfaction.

### Project Presentation

[Loom presentation](https://www.loom.com/share/79a37b885fd748afa663171fc9b74d03?sid=24856dba-cac7-4b5b- 9758-4b844ce4290d)






