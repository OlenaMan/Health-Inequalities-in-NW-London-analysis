
# NW London Health Inequalities Project – Technical Walkthrough

## Data Engineering

### Data Ingestion

The project integrates 30+ public datasets from:

* Office for Health Improvement and Disparities (OHID) Fingertips API
* Office for National Statistics (ONS)
* Greater London Authority (GLA)
* GP Patient Survey (GPPS)
* Ethnicity Facts and Figures

Key Python libraries:

```python
import pandas as pd
import numpy as np
import requests
from pathlib import Path
```
### API Ingestion Example

Data was ingested directly from the OHID Fingertips API.

```python
response = requests.get(api_url)
response.raise_for_status()
data = response.json()
```
```
def fetch_fingertips_indicator(
    indicator_id: int,
    timeout: int = 20,      # fail fast
    retries: int = 1        # keep low on unstable days
) -> pd.DataFrame:
    url = (
        "https://fingertips.phe.org.uk/api/all_data/csv/by_indicator_id"
        f"?indicator_ids={indicator_id}"
    )

    last_exc = None
    for attempt in range(1, retries + 1):
        try:
            r = requests.get(url, timeout=timeout)
            r.raise_for_status()
            return pd.read_csv(pd.io.common.BytesIO(r.content), low_memory=False)
        except Exception as exc:
            last_exc = exc
            if attempt < retries:
                time.sleep(1)
                continue
            raise last_exc
```

Why `requests`?

* Automated API retrieval
* Repeatable data acquisition
* Reduced manual downloads
* Supports scalable pipelines


requests → API call → JSON/data response → Pandas DataFrame → Parquet file



### Why Parquet?

```
PARQUET_ENGINE = "pyarrow" if has_module("pyarrow") else "fastparquet"
print("Parquet engine:", PARQUET_ENGINE)
```

Processed datasets were stored as Parquet files.

Benefits:

* Smaller file sizes
* Faster loading than CSV
* Preserves data types
* Creates a stable analytical layer independent of source APIs

Typical workflow:

```text
API → Pandas → Parquet → Audit → Harmonisation → Modelling
```

---

# Data Processing & Harmonisation

The harmonisation notebook aligned multiple datasets into a consistent analytical structure.

### Borough Standardisation

e.g. "Hammersmith and Fulham" vs. "Hammersmith & Fulham"

```python
dataset_df["area_name"] = (
    dataset_df["area_name"]
    .astype(str)
    .str.strip()
)
```
Purpose:

* Consistent borough naming
* Reliable joins across datasets
* Improved data quality

### Time Standardisation

Different datasets stored time differently.

e.g. model_time:
- 2011 - 13
- 2021
- Year 2019
- 2019/20

```python
df_fingertips_std["model_year"] = pd.to_numeric(
    df_fingertips_std["model_time"]
    .str.extract(r"(\d{4})")[0],
    errors="coerce",
)
```
Purpose:

* Align all datasets to a common analytical timeline

### Data Completeness Audit

```python
completeness_df = pd.DataFrame(
    {
        "total_rows": [len(df_aligned)],
        "rows_with_values": [df_aligned["value"].notna().sum()],
        "rows_missing_values": [df_aligned["value"].isna().sum()],
        "coverage_rate": [
            df_aligned["value"].notna().sum() / len(df_aligned)
        ],
    }
)

completeness_df
```
```
total_rows	rows_with_values	coverage_rate
9157	    5325	            0.581522
```
```
area_name	                coverage_rate
0	Ealing	                0.599665
1	Brent	                0.599332
2	Hillingdon	            0.594915
3	Hounslow                0.593883
4	Harrow	                0.588286
5	Westminster	            0.566396
6	Hammersmith and Fulham	0.564663
7	Kensington and Chelsea	0.537428
```

Purpose:

* Identify missing values
* Assess borough-level coverage
* Evaluate dataset readiness

---

# Health Inequalities Modelling

The modelling notebook transformed multiple indicators into a single framework.

## Z-Score Standardisation

Different indicators operate on different scales:

| Indicator               | Scale            |
| ----------------------- | ---------------- |
| Smoking prevalence      | %                |
| Obesity rate            | %                |
| Hospital admissions     | Rate per 100,000 |
| Healthy Life Expectancy | Years            |

To make indicators comparable:

```python
df_standardised["z_score"] = (
    df_standardised["value"]
    - df_standardised["metric_mean"]
) / df_standardised["metric_std"]
```

Purpose:

* Mean = 0
* Standard deviation = 1
* Enables fair comparison across metrics

---

## Value Direction Alignment

Not all indicators move in the same direction.

Examples:

| Indicator               | Direction |
| ----------------------- | --------- |
| Smoking prevalence      | Adverse   |
| Suicide rate            | Adverse   |
| Healthy Life Expectancy | Positive  |
| Screening coverage      | Positive  |

Direction mapping:

```python
metric_direction_map = {
    "Smoking prevalence": "adverse",
    "Suicide rate": "adverse",
    "HLE male": "positive",
    "HLE female": "positive",
}
```

Adjustment:

```python
df_direction["adjusted_z_score"] = np.where(
    df_direction["direction"] == "adverse",
    df_direction["z_score"],
    -df_direction["z_score"]
)
```

Purpose:

* Align all indicators to the same interpretation
* Higher scores consistently represent worse outcome/higher health burden

---

# Final Outputs

## NW London Borough Inequality Typology

The table below summarises borough-level inequality typologies derived from combined health outcomes, prevention indicators, population structure, and service-access patterns across North West London.

### Interpretation

- Higher inequality scores indicate greater overall inequality burden.
- Preventive inequality score reflects upstream prevention-related risk factors.
- Trend change reflects directional movement over time.
- Recommended actions were generated from borough typology clustering and inequality patterns.

| Borough | Borough Typology | Recommended Action | Inequality Score | Preventive Inequality Score | Black Population Share | Trend Change |
|---|---|---|---:|---:|---:|---:|
| Harrow | Emerging risk | Early intervention and close monitoring | -0.196 | 0.282 | 0.073 | 0.806 |
| Hammersmith and Fulham | High priority: equity + prevention | Targeted outreach, VCSE partnership, prevention | 0.131 | 0.150 | 0.123 | -0.955 |
| Brent | High priority: equity + prevention | Targeted outreach, VCSE partnership, prevention | 0.114 | 0.267 | 0.175 | -1.958 |
| Ealing | High priority: equity + prevention | Targeted outreach, VCSE partnership, prevention | 0.034 | 0.064 | 0.108 | -1.005 |
| Westminster | High priority: outcomes/access | Improve access, pathways, and service responsiveness | 0.009 | -0.454 | 0.081 | -0.684 |
| Hounslow | High priority: prevention | Strengthen early intervention and prevention | 0.147 | 0.154 | 0.072 | -0.691 |
| Hillingdon | Lower burden / monitor | Maintain performance and monitor | -0.033 | -0.107 | 0.078 | -0.414 |
| Kensington and Chelsea | Lower burden / monitor | Maintain performance and monitor | -0.247 | -0.357 | 0.079 | 0.055 |

---

# Final Notes

This project is an iterative project. The part I demonstrated is Phase 1 and I am now working on Phase 3 where I look at more granular geography - from borough level to ward and LSOA level - to provide a more targeted intervention planning.

This project demonstrates an end-to-end data workflow rather than just health analysis. I identified and ingested data from more than 30 sources, standardised and harmonised them into a common analytical structure, and developed a modelling framework to support evidence-based decision making.

### Transferrable skills

The same skills are directly transferable to commercial environments because the core challenge - integrating fragmented data and turning it into actionable insight - is common across healthcare, finance, retail and the public sector.

Retail:
- Combine customer, transaction and loyalty data
- Identify drivers of churn
- Improve customer retention

Financial Services:
- Combine customer, product and risk data
- Detect high-risk segments
- Improve risk management

Public Sector:
- Combine demographic, service usage and deprivation data
- Prioritise communities needing support

### To improve the project further:

- The current project was designed as an analytical research framework. In a production environment, I would convert notebook logic into modular Python scripts orchestrated through an automated pipeline.

- I would also automate the code by placing it on Databricks, and programme Notebook run nightly so outputs arte updated continuously.

- I would focus on predictive modelling to identify emerging inequality risks before outcomes deteriorate.



