# 🚀 Automated ETL Pipeline: Indonesia Regional Economic Data (BPS)

## 📌 Project Overview
Manual collection of regional economic data (PDRB) from the Central Bureau of Statistics (BPS) is often hindered by fragmented APIs and inconsistent regional identifiers. This project automates the **Extraction, Transformation, and Loading (ETL)** of PDRB data across **38 provinces** from 2010 to 2025.

* **Data Source:** BPS (Badan Pusat Statistik) Indonesia Web API.
* **Target:** PostgreSQL Relational Database.
* **Tech Stack:** Python (Pandas, Requests, SQLAlchemy), PostgreSQL, JSON Processing.

---

## 🏗️ Architecture & Pipeline
The pipeline follows a modular architecture to ensure data integrity from the BPS server to the final analytics layer.



1.  **Extraction:** Programmatically fetching nested JSON data from the BPS dynamic API using the `requests` library.
2.  **Transformation:** * Slicing 16-digit composite strings to extract meaningful Year and Value codes.
    * Mapping BPS internal `prov_map` codes (e.g., `9400`) to standardized provincial names.
    * Handling the "Papua Expansion" logic (splitting Papua into New Autonomous Regions/DOB).
3.  **Loading:** Upserting cleaned, long-format data into a **PostgreSQL** schema for downstream BI analysis.

---

## 🔍 Engineering Highlights

### 1. Handling Nested JSON & Metadata
The BPS API returns data in a "Cell" format that isn't immediately tabular. I developed a recursive parser to flatten this structure, ensuring that `region_id` and `pdrb_value` are correctly aligned across 15+ years of historical data.

### 2. Standardizing Regional Identifiers
**Challenge:** BPS metadata uses 4-digit codes (e.g., `1100`) while older maps use 2-digit codes (`11`).
**Solution:** Implemented a string-cleaning layer that standardizes all regional IDs to a 4-digit format, preventing "null joins" during geospatial visualization.

### 3. Case Sensitivity & UI/UX Consistency
To ensure the data is "Dashboard Ready," the pipeline automatically converts ALL CAPS province names to **Title Case** and removes the "National Total" (ID 9999) to prevent double-counting in aggregations.

---

## 🛠️ Code Snippet: Transformation Layer
```python
# Mapping ID to Name and Standardizing Case
def clean_bps_metadata(df, prov_map):
    df['provinsi'] = df['vervar'].map(prov_map)
    df = df[df['region_id'] != '9999'] # Exclude National Total
    df['provinsi'] = df['provinsi'].str.title() # For UI/UX consistency
    return df
