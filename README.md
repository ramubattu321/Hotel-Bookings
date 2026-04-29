# Hotel Booking Data Wrangling & Platform Analysis

---

## Overview

A Python-based data wrangling project that cleans and transforms semi-structured hotel booking data where booking platform names (Hotels, Booking.com, Expedia, Cleartrip) are embedded as marker rows within the dataset. The project extracts platform information using marker row detection and backfilling, then performs platform-level EDA to compare booking distribution across channels.

---

## Business Problem

Raw hotel booking exports often contain mixed-format records — actual bookings interleaved with platform identifier rows. This makes direct analysis impossible without first untangling the structure.

This project solves that by:
- Detecting and extracting platform marker rows embedded in the data
- Assigning the correct booking platform to every booking record using backfilling
- Removing invalid rows to produce a clean, analysis-ready dataset
- Performing platform-level EDA to identify top booking channels

---

## The Data Challenge — Raw vs. Clean

The raw dataset is semi-structured. Booking platform names appear as standalone rows inside the data rather than as a column — making it impossible to analyze by source without wrangling first.

**Raw Data (before wrangling):**
```
Date          Company      Person Name    Room
-----------   ----------   -----------    ----
Expedia       NaN          NaN            NaN      ← platform marker row
2023-01-05    Acme Corp    John Smith     101
2023-01-06    Beta Ltd     Jane Doe       102
Booking.com   NaN          NaN            NaN      ← platform marker row
2023-01-07    Gamma Inc    Bob Lee        201
2023-01-08    Delta Co     Alice Ray      305
```

**Clean Data (after wrangling):**
```
Date          Company      Person Name    Room    Platform
-----------   ----------   -----------    ----    -----------
2023-01-05    Acme Corp    John Smith     101     Expedia
2023-01-06    Beta Ltd     Jane Doe       102     Expedia
2023-01-07    Gamma Inc    Bob Lee        201     Booking.com
2023-01-08    Delta Co     Alice Ray      305     Booking.com
```

---

## Dataset

| Column | Description |
|--------|-------------|
| Date | Booking date (or platform name for marker rows) |
| Company | Company associated with the booking |
| Person Name | Guest name |
| Room | Room number assigned |
| Platform *(derived)* | Booking platform — extracted via wrangling |

**Platforms covered:** Hotels · Booking.com · Expedia · Cleartrip

---

## Methodology

### Step 1 — Marker Row Detection

Identified rows where the `Room` column is null — these are platform marker rows, not actual bookings:

```python
# Identify platform marker rows (no room number = platform label row)
platform_mask = df['Room'].isna()

# Extract platform name from the Date column of marker rows
df['Platform'] = df.loc[platform_mask, 'Date']
```

### Step 2 — Backfilling Platform Labels

Used `bfill()` (backward fill) to propagate platform names forward to all booking rows beneath each marker:

```python
# Backfill platform name down to all bookings below each marker row
df['Platform'] = df['Platform'].bfill()
```

### Step 3 — Remove Marker Rows

Dropped the platform marker rows — keeping only actual booking records:

```python
# Remove marker rows, keep only valid bookings
df_clean = df[df['Room'].notna()].reset_index(drop=True)
```

### Step 4 — Platform-Level EDA

Analyzed and compared booking distribution across all four platforms:

```python
# Booking count per platform
platform_counts = df_clean['Platform'].value_counts()

# Visualize
platform_counts.plot(kind='bar', title='Bookings by Platform')
```

---

## Key Insights

| Platform | Finding |
|----------|---------|
| Expedia | Highest number of bookings across all platforms |
| Booking.com | Second highest — strong corporate segment |
| Hotels | Consistent mid-range volume |
| Cleartrip | Lowest volume — niche or regional channel |

- Platform distribution was hidden in the raw data — invisible without wrangling
- Backfilling correctly assigned platform to 100% of booking records
- Clean dataset enables accurate channel attribution for business reporting

---

## Visualizations

The notebook produces a bar chart comparing total booking counts across Expedia, Booking.com, Hotels, and Cleartrip — highlighting Expedia as the dominant booking channel.

> To view the charts, open `hotel_booking_platform_analysis.ipynb` in Jupyter and run all cells.

---

## Project Structure

```
├── hotel_booking_platform_analysis.ipynb    # Main notebook — wrangling + EDA
└── README.md                                # Project documentation
```

---

## How to Run

```bash
# 1. Clone the repository
git clone https://github.com/ramubattu321/Hotel-Booking-Data-Wrangling-Platform-Analysis.git
cd Hotel-Booking-Data-Wrangling-Platform-Analysis

# 2. Install dependencies
pip install pandas numpy matplotlib jupyter

# 3. Open the notebook
jupyter notebook hotel_booking_platform_analysis.ipynb
```

---

## Business Impact

- **Channel attribution** — correctly assigns every booking to its source platform
- **Platform performance analysis** — identifies top and underperforming booking channels
- **Structured reporting** — transforms unusable raw exports into clean analytical datasets
- **Reusable pipeline** — wrangling logic applicable to any similarly structured booking export
- **Marketing decisions** — insights support budget allocation across OTA (Online Travel Agency) channels

---

## Tools & Technologies

| Tool | Purpose |
|------|---------|
| Python | Core scripting language |
| Pandas | Data wrangling, backfilling, cleaning |
| NumPy | Numerical operations |
| Matplotlib | Bar chart visualization |
| Jupyter Notebook | Interactive analysis environment |

---

## Author
Ramu Battu

**Ramu Battu**
MS in Data Analytics — California State University, Fresno
📧 ramuusa61@gmail.com
