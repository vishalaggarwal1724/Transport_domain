Ad Hoc SQL Queries for Business Analysis:-
Managing a large-scale transportation service requires deep insights into passenger behavior, revenue generation, 
and trip performance. This project focuses on analyzing business-critical metrics using SQL queries to help decision-makers 
optimize operations, enhance passenger retention, and improve financial performance.
----------------------------------------------------------------------------------------------------------------------------------------------------------------------

Database Tables Used:
-fact_passenger_summary: Stores passenger-related data, including new and repeat passengers.
-fact_trips: Contains trip-level details such as fare amount and distance traveled.
-dim_city: Stores city metadata.
-dim_date: Holds date-related details, such as months.
-dim_repeat_trip_distribution: Contains repeat trip counts per city.
-targets_db.monthly_target_trips: Stores target trips per city per month.
-The following queries provide valuable insights for business decisions, helping in optimizing operations, 
 setting better targets, and improving passenger retention strategies.
-----------------------------------------------------------------------------------------------------------------------------------------------------------------------

Overview:
 This repository contains SQL queries designed for analyzing business performance metrics related to passenger trips, fare revenues, 
and target achievements in a transportation service company. These queries help extract meaningful insights into city-level passenger trends, 
trip performance, and revenue contributions.

-----------------------------------------------------------------------------------------------------------------------------------------------------------------------
1.Identify Cities with Highest and Lowest Total New Passengers
WITH ranked_cities AS (
    SELECT 
        dc.city_name,
        SUM(fps.new_passengers) AS total_new_passengers,
        RANK() OVER (ORDER BY SUM(fps.new_passengers) DESC) AS rank_highest,
        RANK() OVER (ORDER BY SUM(fps.new_passengers)) AS rank_lowest
    FROM 
        fact_passenger_summary fps
    JOIN 
        dim_city dc ON fps.city_id = dc.city_id
    GROUP BY 
        dc.city_name
)
SELECT 
    city_name,
    total_new_passengers,
    CASE 
        WHEN rank_highest <= 3 THEN 'Top 3'
        WHEN rank_lowest <= 3 THEN 'Bottom 3'
    END AS city_category
FROM 
    ranked_cities
WHERE 
    rank_highest <= 3 OR rank_lowest <= 3;

Discription:
This query ranks cities based on their total new passengers and identifies the top three and bottom three cities. 
The ranking helps in understanding the cities with the highest and lowest passenger acquisition.
----------------------------------------------------------------------------------------------------------------------------------------------------------------------

2.City-Level Fare and Trip Summary Report
SELECT 
    dc.city_name,
    COUNT(ft.trip_id) AS total_trips,
    AVG(ft.fare_amount / ft.distance_travelled_km) AS avg_fare_per_km,
    AVG(ft.fare_amount) AS avg_fare_per_trip,
    (COUNT(ft.trip_id) / (SELECT COUNT(*) FROM fact_trips)) * 100 AS percentage_contribution_to_total_trips
FROM 
    fact_trips ft
JOIN 
    dim_city dc ON ft.city_id = dc.city_id
GROUP BY 
    dc.city_name;

Discription:
This query provides a city-wise summary of trip data, including:
-Total number of trips
-Average fare per kilometer
-Average fare per trip
-Percentage contribution of each city’s trips to the overall trip count
--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------

3. City-Level Repeat Passenger Trip Frequency Report
SELECT 
    dc.city_name,
    SUM(CASE WHEN dtd.trip_count = 2 THEN dtd.repeat_passenger_count ELSE 0 END) * 100.0 / SUM(dtd.repeat_passenger_count) AS "2-Trips",
    SUM(CASE WHEN dtd.trip_count = 3 THEN dtd.repeat_passenger_count ELSE 0 END) * 100.0 / SUM(dtd.repeat_passenger_count) AS "3-Trips",
    SUM(CASE WHEN dtd.trip_count = 4 THEN dtd.repeat_passenger_count ELSE 0 END) * 100.0 / SUM(dtd.repeat_passenger_count) AS "4-Trips",
    SUM(CASE WHEN dtd.trip_count = 5 THEN dtd.repeat_passenger_count ELSE 0 END) * 100.0 / SUM(dtd.repeat_passenger_count) AS "5-Trips",
    SUM(CASE WHEN dtd.trip_count = 6 THEN dtd.repeat_passenger_count ELSE 0 END) * 100.0 / SUM(dtd.repeat_passenger_count) AS "6-Trips",
    SUM(CASE WHEN dtd.trip_count = 7 THEN dtd.repeat_passenger_count ELSE 0 END) * 100.0 / SUM(dtd.repeat_passenger_count) AS "7-Trips",
    SUM(CASE WHEN dtd.trip_count = 8 THEN dtd.repeat_passenger_count ELSE 0 END) * 100.0 / SUM(dtd.repeat_passenger_count) AS "8-Trips",
    SUM(CASE WHEN dtd.trip_count = 9 THEN dtd.repeat_passenger_count ELSE 0 END) * 100.0 / SUM(dtd.repeat_passenger_count) AS "9-Trips",
    SUM(CASE WHEN dtd.trip_count = 10 THEN dtd.repeat_passenger_count ELSE 0 END) * 100.0 / SUM(dtd.repeat_passenger_count) AS "10-Trips"
FROM 
    dim_repeat_trip_distribution dtd
JOIN 
    dim_city dc ON dtd.city_id = dc.city_id
GROUP BY 
    dc.city_name;

Discription:
This report analyzes repeat passenger frequency in each city. It calculates the percentage of passengers who have taken 2 to 10 trips, 
helping in understanding customer retention and repeat trip behavior.
-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
4. Identify Month with Highest Revenue for Each City.
 WITH monthly_revenue AS (
    SELECT 
        dc.city_name,
        dd.month_name,
        SUM(ft.fare_amount) AS revenue
    FROM 
        fact_trips ft
    JOIN 
        dim_city dc ON ft.city_id = dc.city_id
    JOIN 
        dim_date dd ON ft.date = dd.date
    GROUP BY 
        dc.city_name, dd.month_name
),
ranked_revenue AS (
    SELECT 
        city_name,
        month_name AS highest_revenue_month,
        revenue,
        RANK() OVER (PARTITION BY city_name ORDER BY revenue DESC) AS `rank`
    FROM 
        monthly_revenue
)
SELECT 
    city_name,
    highest_revenue_month,
    revenue,
    (revenue / SUM(revenue) OVER (PARTITION BY city_name)) * 100 AS percentage_contribution
FROM 
    ranked_revenue
WHERE 
    `rank` = 1;

Discription:
This query determines the month with the highest revenue for each city by:
-Summing up fare revenue per city and month
-Ranking months based on revenue within each city
-Calculating each month’s percentage contribution to the city's total revenue
----------------------------------------------------------------------------------------------------------------------------------------------------------------------------

5. MonthIy City-Level Trips Target Performance Report.  
SELECT 
    dc.city_name,
    dd.month_name,
    COUNT(ft.trip_id) AS actual_trips,
    mtt.total_target_trips AS target_trips,
    CASE 
        WHEN COUNT(ft.trip_id) > mtt.total_target_trips THEN 'Above Target'
        ELSE 'Below Target'
    END AS performance_status,
    ((COUNT(ft.trip_id) - mtt.total_target_trips) / mtt.total_target_trips) * 100 AS percentage_difference
FROM 
    fact_trips ft
JOIN 
    dim_city dc ON ft.city_id = dc.city_id
JOIN 
    dim_date dd ON ft.date = dd.date
JOIN 
    targets_db.monthly_target_trips mtt ON ft.city_id = mtt.city_id AND dd.start_of_month = mtt.month
GROUP BY 
    dc.city_name, dd.month_name, mtt.total_target_trips
LIMIT 1000;

Discription:
This query compares actual trips against the target trips for each city and month. It also classifies the performance 
as ‘Above Target’ or ‘Below Target’ based on whether the actual trips exceed the target.
---------------------------------------------------------------------------------------------------------------------------------------------------------------------------

6. Repeat Passenger Rate Analysis 
SELECT 
    dc.city_name,
    dd.month_name,
    fps.total_passengers,
    fps.repeat_passengers,
    (fps.repeat_passengers / fps.total_passengers) * 100 AS monthly_repeat_passenger_rate,
    (SUM(fps.repeat_passengers) OVER (PARTITION BY dc.city_name) / 
     SUM(fps.total_passengers) OVER (PARTITION BY dc.city_name)) * 100 AS city_repeat_passenger_rate
FROM 
    fact_passenger_summary fps
JOIN 
    dim_city dc ON fps.city_id = dc.city_id
JOIN 
    dim_date dd ON fps.month = dd.start_of_month;

Discription:-
This report calculates the repeat passenger rate at the monthly and city level, showing:
-The percentage of passengers who are repeat customers
-The city-wide repeat passenger rate, providing insights into customer loyalty trends
-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------
