# Hotel Booking Data Wrangling & Platform Analysis

## Overview

This project analyzes hotel booking data using **Python and Pandas** to clean semi-structured records and identify booking patterns across different platforms.

The goal of the analysis is to transform messy raw booking data into a structured dataset that can be used for platform-level reporting and operational insights.

## Business Problem

Hotel booking data often contains inconsistent records and embedded text markers that make analysis difficult. In this dataset, some rows represent booking platform labels such as **Hotels, Booking, Expedia, and Cleartrip** instead of actual booking transactions.

This project focuses on cleaning the data and assigning the correct booking source to each reservation so that platform-level booking activity can be analyzed accurately.

## Dataset

The dataset includes the following columns:

- **Date** – Booking date or booking platform marker  
- **Company** – Company associated with the booking  
- **Person Name** – Guest name  
- **Room number** – Assigned room number  

Some rows contain **platform markers instead of booking records**, which require preprocessing before analysis.

## Technologies Used

- Python  
- Pandas  
- NumPy  
- Matplotlib  
- Jupyter Notebook  

## Data Wrangling Steps

The dataset required several preprocessing steps:

1. **Load dataset**
   ```python
   import pandas as pd
   import numpy as np
   import matplotlib.pyplot as plt

  2. Identify platform marker rows
Detected rows with missing room numbers
Used those rows to isolate booking source information
3. Create booking source column
   mask = df["Room number"].isna()
df["Booking Source"] = np.where(mask, df["Date"], np.nan)
4. Backfill booking platforms
   df["Booking Source"].fillna(method="bfill", inplace=True)
5.Remove invalid rows
df_clean = df.dropna(subset=["Company", "Person Name", "Room number"]).copy()
6. Standardize room number format
df_clean["Room number"] = df_clean["Room number"].astype(int)
Analysis Performed
Booking Platform Distribution

The analysis compares booking volume across different booking platforms.

Example output:

Platform	Bookings
Expedia	48
Hotels	39
Booking	24
Travel Agent 007	12
Cleartrip	11
Visualization

A bar chart was created to compare booking counts across platforms.

platform_counts = df_clean["Booking Source"].value_counts()

plt.figure(figsize=(8, 5))
platform_counts.plot(
    kind="bar",
    title="Bookings by Platform",
    ylabel="Number of Bookings",
    xlabel="Booking Source",
    rot=45
)
plt.tight_layout()
plt.show()

This visualization helps identify which booking source generates the highest number of reservations.

Key Insights
Expedia generated the highest number of bookings in the dataset
Booking activity was distributed across multiple channels including Hotels, Booking, Cleartrip, and Travel Agent sources
Data wrangling was necessary to correctly assign booking source information
Cleaned data enabled accurate platform-level analysis from messy raw records
Business Impact
Supports booking channel performance analysis
Helps identify top-performing reservation sources
Improves the usability of semi-structured hotel booking data
Enables more accurate operational reporting
How to Run the Project

1.Clone the repository

git clone https://github.com/ramubattu321/hotel-booking-cancellation-analysis.git

2.Install dependencies

pip install pandas numpy matplotlib

3.Run the Jupyter Notebook

jupyter notebook
Author

Ramu Battu
MS Data Analytics, California State University, Fresno
