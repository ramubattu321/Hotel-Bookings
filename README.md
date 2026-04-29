# Hotel Booking Data Wrangling & Platform Analysis

![Python](https://img.shields.io/badge/Python-3.10+-3776AB?style=flat&logo=python&logoColor=white)
![SQL](https://img.shields.io/badge/SQL-SQLite-003B57?style=flat&logo=sqlite&logoColor=white)
![Pandas](https://img.shields.io/badge/Pandas-Data%20Wrangling-150458?style=flat&logo=pandas&logoColor=white)
![Status](https://img.shields.io/badge/Status-Complete-brightgreen?style=flat)

---

## Overview

An end-to-end data wrangling and SQL analytics project that cleans and transforms semi-structured hotel booking data — where booking platform names are embedded as marker rows within the dataset — and performs platform-level analysis to compare booking performance across **Expedia, Booking.com, Hotels, and Cleartrip**.

---

## Business Problem

Raw hotel booking exports contain mixed-format records where actual bookings are interleaved with platform identifier rows. This makes direct analysis impossible without first untangling the structure. This project:

- Detects and extracts platform marker rows embedded in the data
- Assigns the correct booking platform to every record using backfilling
- Loads the cleaned data into a relational database
- Runs 16 production SQL queries to compare platform performance, cancellations, revenue, and booking patterns

---

## The Data Challenge — Raw vs. Clean

**Raw Data (before wrangling):**
```
Date          Company      Person Name    Room
-----------   ----------   -----------    ----
Expedia       NaN          NaN            NaN      ← platform marker row
2023-01-05    Acme Corp    Guest_0001     101
2023-01-06    Beta Ltd     Guest_0002     102
Booking.com   NaN          NaN            NaN      ← platform marker row
2023-01-07    Gamma Inc    Guest_0003     201
```

**Clean Data (after wrangling):**
```
Date          Company      Guest_Name    Room    Platform      Status
-----------   ----------   ----------    ----    -----------   ---------
2023-01-05    Acme Corp    Guest_0001    101     Expedia       Completed
2023-01-06    Beta Ltd     Guest_0002    102     Expedia       Completed
2023-01-07    Gamma Inc    Guest_0003    201     Booking.com   Completed
```

**Wrangling Logic (Python):**
```python
# Identify platform marker rows — no room number = platform label
platform_mask = df['Room'].isna()

# Extract platform name from Date column of marker rows
df['Platform'] = df.loc[platform_mask, 'Date']

# Backfill platform label down to all bookings below each marker
df['Platform'] = df['Platform'].bfill()

# Remove marker rows — keep only valid bookings
df_clean = df[df['Room'].notna()].reset_index(drop=True)
```

---

## Dataset

**1,200 bookings | 4 platforms | 2023 (Jan–Dec)**

| Column | Description |
|--------|-------------|
| platform | Booking source — Expedia / Booking.com / Hotels / Cleartrip |
| booking_date | Date booking was made |
| check_in_date | Guest check-in date |
| check_out_date | Guest check-out date |
| room_number | Room assigned |
| company | Corporate booking company (NULL for leisure) |
| guest_name | Guest identifier |
| room_type | Standard / Deluxe / Suite / Executive |
| num_guests | Number of guests |
| price_per_night | Nightly rate (USD) |
| total_revenue | Total value (0 if cancelled) |
| status | Confirmed / Completed / Cancelled / No-Show |
| payment_method | Credit Card / PayPal / UPI / etc. |
| country | Guest country of origin |

---

## Project Structure

```
Hotel-Booking-Data-Wrangling-Platform-Analysis/
│
├── hotel_booking_platform_analysis.ipynb   # Python wrangling + EDA
│
├── sql/
│   ├── schema.sql          # Database schema — 2 tables + indexes
│   ├── sample_data.sql     # 1,200 bookings + 4 platform records
│   ├── hotel_queries.sql   # 16 production SQL queries
│   └── setup_and_run.py    # Creates SQLite DB + runs all queries
│
└── README.md
```

---

## SQL Database Schema

```sql
-- Booking records (1,200 rows)
bookings (
    booking_id, platform, booking_date, check_in_date, check_out_date,
    room_number, company, guest_name, room_type, num_guests,
    price_per_night, total_revenue, status, payment_method, country
)

-- Platform metadata (4 rows)
platforms (
    platform_id, platform_name, commission_pct, region, launch_year
)
```

**Platform Reference:**

| Platform | Commission % | Region | Since |
|----------|-------------|--------|-------|
| Expedia | 18.5% | Global | 2001 |
| Booking.com | 15.0% | Global | 1996 |
| Hotels | 12.0% | USA-focused | 2001 |
| Cleartrip | 8.5% | Asia-focused | 2006 |

---

## SQL Queries — 16 Production Queries

### Platform Performance & Market Share

```sql
-- Platform summary with net revenue after commission
SELECT b.platform, p.commission_pct,
    COUNT(*) AS total_bookings,
    ROUND(SUM(b.total_revenue), 2) AS gross_revenue,
    ROUND(SUM(b.total_revenue) * (1 - p.commission_pct/100.0), 2) AS net_revenue
FROM bookings b
JOIN platforms p ON b.platform = p.platform_name
GROUP BY b.platform ORDER BY gross_revenue DESC;

-- Platform market share (SUM OVER window function)
SELECT platform,
    ROUND(100.0 * COUNT(*) / SUM(COUNT(*)) OVER (), 2)          AS booking_share_pct,
    ROUND(100.0 * SUM(total_revenue)
          / SUM(SUM(total_revenue)) OVER (), 2)                  AS revenue_share_pct
FROM bookings GROUP BY platform ORDER BY revenue_share_pct DESC;
```

### Window Functions

```sql
-- Cumulative revenue per platform over time (SUM OVER)
SELECT platform, booking_date,
    ROUND(SUM(SUM(total_revenue)) OVER (
        PARTITION BY platform ORDER BY booking_date), 2) AS cumulative_revenue
FROM bookings GROUP BY platform, booking_date;

-- Month-over-month growth using LAG
WITH monthly AS (
    SELECT platform, STRFTIME('%Y-%m', booking_date) AS month,
           ROUND(SUM(total_revenue), 2) AS revenue
    FROM bookings GROUP BY platform, month
)
SELECT platform, month, revenue,
    ROUND(100.0*(revenue - LAG(revenue) OVER (PARTITION BY platform ORDER BY month))
          / NULLIF(LAG(revenue) OVER (PARTITION BY platform ORDER BY month), 0), 1) AS mom_growth_pct
FROM monthly;

-- High-value bookings — top 10% using NTILE
WITH ranked AS (
    SELECT *, NTILE(10) OVER (ORDER BY total_revenue DESC) AS decile
    FROM bookings WHERE status != 'Cancelled'
)
SELECT platform, room_type,
    COUNT(*) AS high_value_bookings,
    ROUND(AVG(total_revenue), 2) AS avg_revenue
FROM ranked WHERE decile = 1
GROUP BY platform, room_type ORDER BY avg_revenue DESC;
```

### All 16 Queries Summary

| # | Query | SQL Technique |
|---|-------|--------------|
| 1 | Platform performance summary | JOIN, GROUP BY, SUM |
| 2 | Platform market share | SUM OVER window |
| 3 | Cancellation rate by platform | CASE WHEN, GROUP BY |
| 4 | Room type analysis by platform | Filtered GROUP BY |
| 5 | Monthly booking trend | STRFTIME, GROUP BY |
| 6 | Monthly revenue pivot | CASE WHEN pivot |
| 7 | Cumulative revenue over time | SUM OVER window |
| 8 | Top platform per day | RANK OVER window + CTE |
| 9 | Lead time analysis (days to check-in) | JULIANDAY date math |
| 10 | Length of stay by room type | JULIANDAY date math |
| 11 | Top 10 countries by revenue | SUM OVER window |
| 12 | Payment method distribution | PARTITION BY window |
| 13 | High-value bookings top 10% | NTILE window + CTE |
| 14 | Month-over-month growth | LAG window + CTE |
| 15 | Corporate vs leisure bookings | CASE WHEN segmentation |
| 16 | Platform efficiency scorecard | Nested CTE + RANK |

---

## Key Results

| Platform | Bookings | Gross Revenue | Net Revenue | Cancel Rate |
|----------|----------|--------------|-------------|------------|
| **Expedia** | 439 (37%) | $349,465 | $284,814 | 14.4% |
| **Booking.com** | 379 (32%) | $288,176 | $244,949 | 18.5% |
| **Hotels** | 236 (20%) | $176,183 | $155,041 | 18.6% |
| **Cleartrip** | 146 (12%) | $100,359 | $91,829 | 18.5% |

**Key Insights:**
- **Expedia** generates the highest bookings and gross revenue — dominant platform
- **Cleartrip** has the lowest commission (8.5%) — most hotel-friendly per booking
- **Expedia** has the lowest cancellation rate (14.4%) — most reliable bookings
- **Suite** rooms on Expedia generate the highest single-category revenue ($111K)
- **Corporate bookings** have a higher average order value but slightly higher cancellation rate

---

## How to Run

```bash
# 1. Clone the repository
git clone https://github.com/ramubattu321/Hotel-Booking-Data-Wrangling-Platform-Analysis.git
cd Hotel-Booking-Data-Wrangling-Platform-Analysis

# 2. Install dependencies
pip install pandas numpy matplotlib jupyter

# 3. Create SQLite database and run all 16 SQL queries
python sql/setup_and_run.py

# 4. Open Python wrangling notebook
jupyter notebook hotel_booking_platform_analysis.ipynb

# 5. Run SQL queries directly (SQLite CLI)
sqlite3 sql/hotel_bookings.db < sql/hotel_queries.sql
```

---

## Tools & Technologies

| Tool | Purpose |
|------|---------|
| Python | Data wrangling — marker detection + backfilling |
| Pandas | Data cleaning, transformation, EDA |
| SQL (SQLite) | Platform analytics — 16 production queries |
| NumPy | Numerical operations |
| Matplotlib | Bar charts and booking trend visualizations |
| Jupyter Notebook | Interactive wrangling environment |

---

## Author

**Ramu Battu**
MS in Data Analytics — California State University, Fresno
📧 ramuusa61@gmail.com
