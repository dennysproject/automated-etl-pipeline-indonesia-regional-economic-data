# 🚀 Automated ETL Pipeline: Indonesia Regional Economic Data (BPS)

## 📌 Project Overview

The BPS (Badan Pusat Statistik) Web API returns regional economic data as nested JSON keyed by composite IDs, which makes it hard to use directly for analysis. This project extracts regional PDRB (Gross Regional Domestic Product) data for 2010 to 2025, transforms it into a clean long-format table, and loads it into PostgreSQL for BI analysis.

- **Data Source:** BPS (Badan Pusat Statistik) Indonesia Web API
- **Coverage:** Regency and city level data, grouped under up to 38 provinces, 2010 to 2025
- **Target:** PostgreSQL relational database
- **Tech Stack:** Python (Pandas, Requests, SQLAlchemy), PostgreSQL, JSON processing

---

## 🏗️ Pipeline

```
BPS Web API (JSON)  ->  Python (requests, pandas)  ->  PostgreSQL  ->  BI tools
```

1. **Extraction:** Request the BPS API once per year (year IDs 110 to 125, which map to 2010 to 2025) using `requests`, with a 1 second pause between calls to avoid rate limits.
2. **Transformation:**
   - Match each value in the JSON `datacontent` to its region and year by rebuilding its composite ID (region + variable + turvar + year + turtahun).
   - Derive the province from the first 2 digits of each region ID and map it to a province name, including the newer Papua provinces.
   - Convert values to numbers and remove rows with missing values or unmapped provinces.
3. **Loading:** Write the cleaned long-format table to PostgreSQL with pandas `to_sql` (full refresh on every run).

---

## 🔍 Engineering Highlights

### 1. Handling Nested JSON

The API response contains three parts: `vervar` (regions), `tahun` (years) and `datacontent` (values keyed by one long composite ID). I rebuilt the expected ID for every region and year combination and used it to attach the right value to the right region and year across 16 years of data.

### 2. Regional Codes and the Papua Expansion

Province names are derived from the first 2 digits of each region ID using a `prov_map` dictionary. The map includes the newer provinces created from Papua (Papua Selatan, Papua Tengah, Papua Pegunungan and Papua Barat Daya), so recent years are labeled correctly.

### 3. Data Cleaning

Values are converted to numbers, and rows with missing values or unmapped province codes (labeled `Unknown (xx)`) are removed before loading.

### 4. Credentials Kept Out of the Code

The API key and database credentials are read from a local `.env` file, which is excluded from Git. A `.env.example` file shows the required variables.

---

## ⚠️ Known Limitations and Next Steps

- **Full refresh:** each run replaces the whole table (`if_exists='replace'`). An incremental load with upserts would be more efficient.
- **ID matching is simple but slow:** for every value, the code checks every region and year combination. Parsing the composite ID directly would be faster.
- **Basic error handling:** failed requests are printed and skipped, with no retry logic.

---

## 🛠️ Code Snippets

**Province mapping and cleaning**

```python
def get_prov_from_id(kab_id):
    kode_prov = str(kab_id)[:2]
    return prov_map.get(kode_prov, f"Unknown ({kode_prov})")

df_final_clean = df_final[
    ~df_final['Provinsi'].str.startswith("Unknown") &
    df_final['PDRB'].notnull()
].copy()
```

**Matching values by composite ID**

```python
expected_id = f"{k_id}21941550{t_id}0"
if full_id == expected_id:
    all_data.append({...})
```

---

## 📊 Sample Output (PostgreSQL table `pdrb_regional_2010_2025`)

The data is stored in long format, with one row per region and year.

| Provinsi | Kabupaten         | Tahun |     PDRB |
| :------- | :---------------- | :---- | -------: |
| Aceh     | Bireuen           | 2025  | 12191.19 |
| Aceh     | Aceh Barat Daya   | 2025  |  3596.03 |
| Aceh     | Aceh Barat        | 2025  |  9068.13 |
| Aceh     | Gayo Lues         | 2025  |  2248.44 |
| Aceh     | Kota Subulussalam | 2025  |  1685.68 |

Quick check in SQL (column names have capital letters, so they need double quotes):

```sql
SELECT "Tahun", COUNT(DISTINCT "Provinsi") AS provinces, COUNT(*) AS rows
FROM pdrb_regional_2010_2025
GROUP BY "Tahun"
ORDER BY "Tahun";
```

---

## 🚀 How to Run

1. **Clone the repository**

```bash
   git clone https://github.com/dennysproject/automated-etl-pipeline-indonesia-regional-economic-data.git
   cd automated-etl-pipeline-indonesia-regional-economic-data
```

2. **Install dependencies** (Python 3.9+)

```bash
   pip install -r requirements.txt
   pip install jupyter
```

3. **Create the database**
   - Create a PostgreSQL database named `bps_analysis`. The notebook creates the table automatically.
4. **Configure credentials**
   - Get an API key from the BPS WebAPI site (webapi.bps.go.id).
   - Copy `.env.example` to `.env` and fill in your API key and database credentials.
5. **Run**
   - Open `etl-pdrb-total.ipynb` and run all cells.

---

**Author**: Denny Ruchiat

**LinkedIn**: https://www.linkedin.com/in/dennyruchiatatwork/
