# BigQuery_businesses_hour_missmatch_soln

## Analyzing the Impact of Business Hour Mismatch on Order Volume in the Food Delivery Industry

## Overview

In the food delivery industry, consistent business hours across platforms are critical to avoid operational inefficiencies. This case study investigates mismatches in business hours between UberEats and Grubhub, focusing on whether the hours listed on Grubhub fall within the operational hours specified for UberEats, which we consider the ground truth.

## Objective

To compute a metric called Business Hour Mismatch between UberEats and Grubhub for a restaurant.

## Dataset 
### UberEats Hours Table (virtual_kitchen_ubereats_hours)

Contains:
- b_name (Restaurant Name)
- vb_name (Virtual Business Name - unique store identifier)
- response (JSON data with hours of operation)

### Grubhub Hours Table (virtual_kitchen_grubhub_hours)

Contains:
- b_name (Restaurant Name)
- vb_name (Virtual Business Name - unique store identifier)
- response (JSON data with hours of operation)

### Key Join: The tuple (b_name, vb_name) serves as the key to map records across both tables.

## SQL Solution

## Step 1: Extract Relevant Business Hours

UberEats Hours
```
WITH Uber AS (
  SELECT
    vb_name AS Uber_Eats_slug,
    b_name,
    startTime AS uber_start_time,
    endTime AS uber_end_time,
    CONCAT(startTime, ' - ', endTime) AS Uber_Eats_Business_Hours
  FROM (
    SELECT
      b_name,
      vb_name,
      JSON_EXTRACT_SCALAR(regularHours, '$.startTime') AS startTime,
      JSON_EXTRACT_SCALAR(regularHours, '$.endTime') AS endTime
    FROM arboreal-vision-339901.take_home_v2.virtual_kitchen_ubereats_hours,
    UNNEST(JSON_EXTRACT_ARRAY(JSON_EXTRACT(response, '$.data.menus.26bd579e-5664-4f0a-8465-2f5eb5fbe705.sections[0].regularHours'))) AS regularHours
  )
)
```

Grubhub Hours

```
, Grub AS (
  SELECT
    vb_name AS Grubhub_slug,
    b_name,
    day,
    startTime AS grub_start_time,
    endTime AS grub_end_time,
    CONCAT(startTime, ' - ', endTime) AS Virtual_Restaurant_Business_Hours
  FROM (
    SELECT
      b_name,
      vb_name,
      JSON_EXTRACT_SCALAR(regularHours, '$.days_of_week[0]') AS day,
      JSON_EXTRACT_SCALAR(regularHours, '$.from') AS startTime,
      JSON_EXTRACT_SCALAR(regularHours, '$.to') AS endTime
    FROM arboreal-vision-339901.take_home_v2.virtual_kitchen_grubhub_hours,
    UNNEST(JSON_EXTRACT_ARRAY(JSON_EXTRACT(response, '$.today_availability_by_catalog.STANDARD_DELIVERY'))) AS regularHours
  )
)
```

# 3 Step 2: Compute Business Hour Mismatch

```
SELECT DISTINCT 
  g.Grubhub_slug,
  g.Virtual_Restaurant_Business_Hours,
  u.Uber_Eats_slug,
  u.Uber_Eats_Business_Hours,
  CASE
    WHEN g.grub_start_time >= u.uber_start_time AND g.grub_end_time <= u.uber_end_time THEN 'In Range'
    WHEN ABS(TIMESTAMP_DIFF(TIMESTAMP(g.grub_start_time), TIMESTAMP(u.uber_start_time), MINUTE)) <= 5
         OR ABS(TIMESTAMP_DIFF(TIMESTAMP(g.grub_end_time), TIMESTAMP(u.uber_end_time), MINUTE)) <= 5 THEN 'Out of Range with 5 mins difference between GH and UE'
    ELSE 'Out of Range'
  END AS is_out_of_range
FROM Uber u, Grub g
WHERE u.b_name = g.b_name 
  AND g.Grubhub_slug = u.Uber_Eats_slug
```
  
## Expected Output
| Grubhub_slug	| Virtual_Restaurant_Business_Hours	| Uber_Eats_slug	| Uber_Eats_Business_Hours	| is_out_of_range                     |
|---------------|-----------------------------------|-----------------|---------------------------|-------------------------------------|
| johnspizz	    | 10:00 - 22:00	                    | johnspizz	      | 10:00 - 22:00	            | In Range                            |
| happyburger	  | 11:00 - 20:00	                    | happyburger	    | 11:00 - 21:00	            | Out of Range with 5 mins difference |
| fastchow	    | 09:00 - 19:00	                    | fastchow	      | 10:00 - 22:00	            | Out of Range                        |

## How It Works

1. UberEats Table: Extracts store hours from the first menu and section using JSON parsing.
2. Grubhub Table: Extracts daily availability hours for standard delivery using JSON parsing.

## Conclusion

This approach provides a systematic way to identify business hour mismatches between UberEats and Grubhub. Ensuring consistency across platforms improves operational efficiency and customer satisfaction.
