# Shipment Data Cleaning Project using BigQuery

## Project Overview

This project focuses on cleaning and validating a logistics shipment dataset using SQL in BigQuery. The dataset contains shipment records with multiple data quality issues such as:

* Duplicate records
* Missing values
* Inconsistent text formatting
* Invalid dates
* Negative shipment weights
* Incorrect delivery timelines
* Mixed capitalization

The goal of this project is to simulate a real-world data cleaning workflow used by data analysts and data engineers before performing reporting or analytics.

---

# Dataset Information

### File Name

`dirty_shipments.csv`

### Total Records

25 rows

### Columns

| Column Name       | Description                  |
| ----------------- | ---------------------------- |
| shipment_id       | Unique shipment identifier   |
| origin_warehouse  | Shipment source warehouse    |
| destination_city  | Destination city             |
| destination_state | Destination state            |
| carrier           | Shipping company             |
| ship_date         | Date shipment was dispatched |
| delivery_date     | Date shipment was delivered  |
| weight_kg         | Shipment weight              |
| freight_cost      | Transportation cost          |
| shipment_status   | Current shipment status      |
| items_count       | Number of items              |
| damage_reported   | Whether damage was reported  |

---

# Project Objectives

This project will help you learn:

* Detecting duplicate records
* Handling NULL values
* Standardizing categorical values
* Validating date logic
* Cleaning numerical columns
* Creating clean analytical datasets
* Generating operational insights

---

# Step 1 — Load the Dataset into BigQuery

```sql
CREATE OR REPLACE TABLE logistics.dirty_shipments (
    shipment_id STRING,
    origin_warehouse STRING,
    destination_city STRING,
    destination_state STRING,
    carrier STRING,
    ship_date STRING,
    delivery_date STRING,
    weight_kg FLOAT64,
    freight_cost FLOAT64,
    shipment_status STRING,
    items_count INT64,
    damage_reported STRING
);
```

---

# Small Level Data Cleaning Questions

# Question 1 — Find Duplicate Shipment Records

## SQL Query

```sql
SELECT shipment_id,
       COUNT(*) AS duplicate_count
FROM logistics.dirty_shipments
GROUP BY shipment_id
HAVING COUNT(*) > 1;
```

## Explanation

This query identifies duplicate shipment IDs by grouping records and counting occurrences.

* `GROUP BY shipment_id` groups identical shipment IDs.
* `HAVING COUNT(*) > 1` filters only duplicates.

## Insight

Duplicate shipment records can cause:

* Incorrect revenue calculations
* Duplicate reporting
* Incorrect inventory tracking

The dataset contains duplicate shipment entries that must be removed before analysis.

---

# Question 2 — Identify Missing Values

## SQL Query

```sql
SELECT *
FROM logistics.dirty_shipments
WHERE ship_date IS NULL
   OR delivery_date IS NULL
   OR damage_reported IS NULL;
```

## Explanation

This query identifies rows containing missing values in important operational columns.

NULL values reduce data reliability and can affect dashboards and KPIs.

## Insight

The dataset contains missing shipment dates and missing damage information.

These records require:

* Data correction
* Imputation
* Business validation

---

# Question 3 — Standardize Warehouse Names

## SQL Query

```sql
SELECT DISTINCT origin_warehouse,
       UPPER(TRIM(origin_warehouse)) AS standardized_warehouse
FROM logistics.dirty_shipments;
```

## Explanation

This query cleans warehouse names using:

* `TRIM()` to remove extra spaces
* `UPPER()` to standardize capitalization

Example:

* `warehouse a`
* `Warehouse A`

Both become:

`WAREHOUSE A`

## Insight

Inconsistent text formatting creates inaccurate grouping and reporting.

Standardization ensures consistent operational analytics.

---

# Question 4 — Find Negative or Invalid Weights

## SQL Query

```sql
SELECT shipment_id,
       weight_kg
FROM logistics.dirty_shipments
WHERE weight_kg <= 0;
```

## Explanation

Shipment weight cannot logically be zero or negative.

This query identifies invalid shipment weights.

## Insight

Negative shipment weights indicate:

* Data entry errors
* ETL issues
* Incorrect sensor values

These records should be corrected or removed.

---

# Question 5 — Standardize Shipment Status

## SQL Query

```sql
SELECT DISTINCT shipment_status,
       INITCAP(TRIM(shipment_status)) AS cleaned_status
FROM logistics.dirty_shipments;
```

## Explanation

The query standardizes shipment statuses by:

* Removing spaces
* Fixing capitalization

Examples:

* `delivered`
* `Delivered`

Both become:

`Delivered`

## Insight

Inconsistent status values can break BI dashboards and KPI reports.

---

# Intermediate Level Data Cleaning Questions

# Question 6 — Convert String Dates into Proper Date Format

## SQL Query

```sql
SELECT shipment_id,
       PARSE_DATE('%Y-%m-%d', ship_date) AS cleaned_ship_date
FROM logistics.dirty_shipments
WHERE REGEXP_CONTAINS(ship_date, r'^\\d{4}-\\d{2}-\\d{2}$');
```

## Explanation

The dataset contains inconsistent date formats.

This query converts string dates into BigQuery DATE format using `PARSE_DATE()`.

## Insight

Standardized dates are necessary for:

* Delivery tracking
* SLA reporting
* Time-series analysis

---

# Question 7 — Find Deliveries Completed Before Shipping

## SQL Query

```sql
SELECT shipment_id,
       ship_date,
       delivery_date
FROM logistics.dirty_shipments
WHERE PARSE_DATE('%Y-%m-%d', delivery_date)
    < PARSE_DATE('%Y-%m-%d', ship_date);
```

## Explanation

Delivery dates should always occur after shipment dates.

This query identifies logically incorrect shipment timelines.

## Insight

Such records indicate:

* Manual entry mistakes
* System synchronization problems
* Incorrect API ingestion

---

# Question 8 — Remove Duplicate Records Using ROW_NUMBER()

## SQL Query

```sql
WITH duplicates_removed AS (
    SELECT *,
           ROW_NUMBER() OVER(
               PARTITION BY shipment_id
               ORDER BY shipment_id
           ) AS rn
    FROM logistics.dirty_shipments
)
SELECT *
FROM duplicates_removed
WHERE rn = 1;
```

## Explanation

This query:

1. Assigns row numbers to duplicate shipment IDs
2. Keeps only the first occurrence
3. Removes duplicate rows

## Insight

`ROW_NUMBER()` is commonly used in production ETL pipelines for deduplication.

---

# Question 9 — Create a Fully Cleaned Dataset

## SQL Query

```sql
CREATE OR REPLACE TABLE logistics.cleaned_shipments AS
SELECT DISTINCT
    shipment_id,
    UPPER(TRIM(origin_warehouse)) AS origin_warehouse,
    INITCAP(TRIM(destination_city)) AS destination_city,
    UPPER(TRIM(destination_state)) AS destination_state,
    INITCAP(TRIM(carrier)) AS carrier,
    SAFE_CAST(weight_kg AS FLOAT64) AS weight_kg,
    SAFE_CAST(freight_cost AS FLOAT64) AS freight_cost,
    INITCAP(TRIM(shipment_status)) AS shipment_status,
    items_count,
    INITCAP(TRIM(damage_reported)) AS damage_reported
FROM logistics.dirty_shipments
WHERE weight_kg > 0;
```

## Explanation

This query performs:

* Deduplication
* Standardization
* Data type cleaning
* Removal of invalid weights
* Formatting improvements

## Insight

A cleaned dataset improves:

* Reporting accuracy
* Dashboard consistency
* Decision-making reliability

---

# Question 10 — Calculate Delivery Duration

## SQL Query

```sql
SELECT shipment_id,
       DATE_DIFF(
           PARSE_DATE('%Y-%m-%d', delivery_date),
           PARSE_DATE('%Y-%m-%d', ship_date),
           DAY
       ) AS delivery_days
FROM logistics.dirty_shipments;
```

## Explanation

This query calculates shipment delivery time in days.

`DATE_DIFF()` helps measure operational efficiency.

## Insight

Delivery duration analysis helps logistics companies:

* Optimize routes
* Monitor SLA performance
* Improve customer satisfaction

---

# Final Insights from the Project

## Key Data Quality Issues Identified

| Issue                       | Impact                      |
| --------------------------- | --------------------------- |
| Duplicate records           | Incorrect reporting         |
| Missing values              | Reduced reliability         |
| Inconsistent capitalization | Fragmented analytics        |
| Invalid dates               | Incorrect timeline analysis |
| Negative weights            | Operational inaccuracies    |
| Mixed status values         | Dashboard inconsistency     |

---

# Business Insights

After cleaning the dataset:

* Shipment tracking becomes more accurate
* Delivery KPIs become reliable
* Warehouses can be analyzed consistently
* Carrier performance can be measured correctly
* Operational anomalies become easier to detect

---

# Conclusion

This project demonstrates how SQL and BigQuery can be used to clean operational logistics data.

The workflow reflects real-world industry practices used in:

* Supply chain analytics
* Logistics reporting
* ETL pipelines
* Data warehousing
* Business intelligence systems

The cleaned dataset can now be safely used for:

* Dashboarding
* KPI reporting
* Predictive analytics
* Operational monitoring
* Machine learning pipelines
  
Author
Shatabdi Dutta
