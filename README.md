# Banking_analytics_dashboard
# Banking Analytics Suite: Churn, Segmentation, Fraud & Credit Risk

A Python + SQL + Power BI project analyzing customer churn, segmentation, transaction fraud, and loan default risk across 294,853 real banking records — built to identify *where* risk and value concentrate, and *what a bank should act on first*.

---

## Overview

This project takes three real, independently-sourced banking datasets and turns them into a decision-ready analytics system. Each dataset was cleaned with Python (pandas), loaded into a SQLite database alongside 3 supporting dimension tables, queried with 20 SQL scripts spanning Basic, Intermediate, Advanced, and Very Advanced difficulty, then connected to Power BI to build a 5-page interactive report (4 dashboards + an executive summary) with custom DAX measures, slicers, and data-driven recommendations.

---

## STAR

**Situation:** A bank has no unified view of where it's losing customers, where fraud risk concentrates, or which loans are most likely to default — each function (retention, fraud, credit) traditionally operates in its own silo.

**Task:** Build an end-to-end analytics suite — from raw data to a decision-ready dashboard — that surfaces churn drivers, high-value segments, fraud patterns, and credit risk concentration in one place.

**Action:** Cleaned and validated 3 real datasets in Python (pandas), engineered features including a churn flag, hour-of-day fraud timing, and risk labels, then loaded everything into a SQLite database alongside 3 purpose-built dimension tables (card tier, loan purpose category, time-of-day shift). Wrote 20 SQL queries progressing from basic aggregation through simple joins, window functions (`RANK`, `PERCENT_RANK`, `NTILE`), correlated subqueries, and CTEs. Connected the results to Power BI and built a 5-page report using DAX measures (`CALCULATE`, `DIVIDE`, `ALL`) for churn risk indexing, segment value share, and fraud rate indexing.

**Result:** Identified that low-spend customers churn at nearly 2x the overall rate (30.78% vs 16.07%), found that 2,754 customers (27% of the base) qualify as high-value, detected that fraud transactions average 40% higher in value than legitimate ones, and confirmed that 36% of total loan value sits in high-risk accounts despite those accounts making up only 30% of applicants.

---

## Dataset

This project uses **3 real datasets**, each verified against known public statistics (row counts, fraud counts, default rates) rather than assumed to be authentic.

| Dataset | Source | Records | Powers |
|---|---|---|---|
| **Credit Card Customers** | [Kaggle — Sakshi Goyal](https://www.kaggle.com/datasets/sakshigoyal7/credit-card-customers) | 10,127 customers | Churn & Retention, Segmentation |
| **Credit Card Fraud Detection** | [Kaggle — ULB](https://www.kaggle.com/datasets/mlg-ulb/creditcardfraud) | 284,807 transactions (473 confirmed fraud after dedup) | Fraud Detection |
| **German Credit Risk** | [Kaggle/UCI — Statlog](https://www.kaggle.com/datasets/uciml/german-credit) | 1,000 loan applicants | Credit Risk |

Real banks don't publicly release a single merged dataset spanning churn, fraud, and credit risk for privacy/regulatory reasons — this project follows standard industry-portfolio practice: multiple real, verified datasets, each covering one analytical module, unified under one dashboard suite and one narrative.

---

## Data Cleaning (Python — pandas)

Raw data required real cleaning: 1,081 duplicate rows in the fraud dataset, a raw seconds-elapsed time column with no analytical value, and Kaggle-documented artifact columns left over from an unrelated model. Full script: [`python/clean_data.py`](python/clean_data.py)

### Basic Cleaning

```python
import pandas as pd

# Drop leftover Naive_Bayes_* columns — a documented Kaggle artifact, not source data
nb_cols = [c for c in df.columns if c.startswith("Naive_Bayes")]
df = df.drop(columns=nb_cols, errors="ignore")

# Remove exact duplicate transactions from the fraud dataset
df = df.drop_duplicates()

# Data integrity checks
assert df["CLIENTNUM"].is_unique, "Duplicate customer IDs found"
assert df.isnull().sum().sum() == 0, "Unexpected nulls found"
assert df.duplicated().sum() == 0, "Unexpected duplicates found"
```

### Advanced Cleaning

```python
import sqlite3

# Engineer Hour_of_Day from raw seconds-elapsed Time column
df["Hour_of_Day"] = (df["Time"] % 86400) // 3600
df["Hour_of_Day"] = df["Hour_of_Day"].astype(int)

# Feature engineering: binary churn flag, readable risk labels
df["Churned"] = (df["Attrition_Flag"] == "Attrited Customer").astype(int)
df["Class_Label"] = df["Class"].map({0: "Legitimate", 1: "Fraud"})
df["Risk_Label"] = df["credit_risk"].map({1: "Good", 0: "Bad"})

# Build a dimension table (star schema) — mirrors the SQL JOIN pattern in pandas
card_tier_dim = pd.DataFrame({
    "Card_Category": ["Blue", "Silver", "Gold", "Platinum"],
    "Tier_Rank": [1, 2, 3, 4],
    "Annual_Fee": [0, 95, 195, 495],
})

# Load fact + dimension tables into SQLite for downstream SQL analysis
conn = sqlite3.connect("bank_analytics.db")
df.to_sql("customers", conn, if_exists="replace", index=False)
card_tier_dim.to_sql("card_tier_dim", conn, if_exists="replace", index=False)
conn.close()
```

---

## Business Problems and Solutions

### Basic Queries

**1. What is the overall customer churn rate?**

```sql
SELECT
    COUNT(*) AS total_customers,
    SUM(Churned) AS churned_customers,
    ROUND(100.0 * SUM(Churned) / COUNT(*), 2) AS churn_rate_pct
FROM customers;
```

**2. What is the overall fraud rate across all transactions?**

```sql
SELECT
    COUNT(*) AS total_transactions,
    SUM(Class) AS fraud_count,
    ROUND(100.0 * SUM(Class) / COUNT(*), 3) AS fraud_rate_pct
FROM fraud_transactions;
```

**3. What is the overall loan default rate?**

```sql
SELECT
    COUNT(*) AS total_applicants,
    SUM(CASE WHEN credit_risk = 0 THEN 1 ELSE 0 END) AS bad_credit_count,
    ROUND(100.0 * SUM(CASE WHEN credit_risk = 0 THEN 1 ELSE 0 END) / COUNT(*), 2) AS default_rate_pct
FROM credit_applicants;
```

**4. How many customers qualify as high-value (credit limit > $10K)?**

```sql
SELECT
    COUNT(*) AS high_value_customers,
    ROUND(100.0 * COUNT(*) / (SELECT COUNT(*) FROM customers), 2) AS pct_of_base
FROM customers
WHERE Credit_Limit > 10000;
```

**5. What is the average loan amount and duration across all applicants?**

```sql
SELECT
    ROUND(AVG(amount), 2) AS avg_loan_amount,
    ROUND(AVG(duration), 1) AS avg_duration_months
FROM credit_applicants;
```

### Intermediate Queries (Simple Joins)

**6. What is the churn rate by card tier, including annual fee context?**

```sql
SELECT
    d.Card_Category, d.Annual_Fee,
    COUNT(*) AS customers,
    ROUND(100.0 * SUM(c.Churned) / COUNT(*), 2) AS churn_rate_pct
FROM customers c
JOIN card_tier_dim d ON d.Card_Category = c.Card_Category
GROUP BY d.Card_Category, d.Annual_Fee
ORDER BY d.Annual_Fee;
```

**7. How does average credit limit and spend compare by card tier?**

```sql
SELECT
    d.Card_Category, d.Tier_Rank,
    ROUND(AVG(c.Credit_Limit), 2) AS avg_credit_limit,
    ROUND(AVG(c.Total_Trans_Amt), 2) AS avg_transaction_amount
FROM customers c
JOIN card_tier_dim d ON d.Card_Category = c.Card_Category
GROUP BY d.Card_Category, d.Tier_Rank
ORDER BY d.Tier_Rank;
```

**8. What is the default rate by loan purpose category (grouped)?**

```sql
SELECT
    d.purpose_category,
    COUNT(*) AS applicants,
    ROUND(100.0 * SUM(CASE WHEN ca.credit_risk = 0 THEN 1 ELSE 0 END) / COUNT(*), 2) AS default_rate_pct
FROM credit_applicants ca
JOIN loan_purpose_dim d ON d.purpose = ca.purpose
GROUP BY d.purpose_category
ORDER BY default_rate_pct DESC;
```

**9. What is the average loan amount by purpose category?**

```sql
SELECT
    d.purpose_category,
    COUNT(*) AS applicants,
    ROUND(AVG(ca.amount), 2) AS avg_loan_amount
FROM credit_applicants ca
JOIN loan_purpose_dim d ON d.purpose = ca.purpose
GROUP BY d.purpose_category
ORDER BY avg_loan_amount DESC;
```

**10. What is the fraud rate and transaction volume by time-of-day shift?**

```sql
SELECT
    d.Shift,
    COUNT(*) AS total_transactions,
    SUM(f.Class) AS fraud_count,
    ROUND(100.0 * SUM(f.Class) / COUNT(*), 3) AS fraud_rate_pct
FROM fraud_transactions f
JOIN hour_shift_dim d ON d.Hour_of_Day = f.Hour_of_Day
GROUP BY d.Shift
ORDER BY fraud_rate_pct DESC;
```

### Advanced Queries (Window Functions & Subqueries)

**11. Rank card categories by churn rate**

```sql
SELECT
    Card_Category,
    ROUND(100.0 * SUM(Churned) / COUNT(*), 2) AS churn_rate_pct,
    RANK() OVER (ORDER BY 1.0 * SUM(Churned) / COUNT(*) DESC) AS churn_rank
FROM customers
GROUP BY Card_Category;
```

**12. Where does each customer's spend sit within their income category?**

```sql
SELECT
    CLIENTNUM,
    Income_Category,
    Total_Trans_Amt,
    ROUND(PERCENT_RANK() OVER (PARTITION BY Income_Category ORDER BY Total_Trans_Amt) * 100, 1) AS spend_percentile
FROM customers
ORDER BY Income_Category, Total_Trans_Amt DESC;
```

**13. Which customers spend more than the average for their card category?**

```sql
SELECT
    e.CLIENTNUM,
    e.Card_Category,
    e.Total_Trans_Amt
FROM customers e
WHERE e.Total_Trans_Amt > (
    SELECT AVG(e2.Total_Trans_Amt)
    FROM customers e2
    WHERE e2.Card_Category = e.Card_Category
);
```

**14. What are the top 5 riskiest hours of day, ranked by fraud rate?**

```sql
SELECT Hour_of_Day, fraud_rate_pct FROM (
    SELECT
        Hour_of_Day,
        ROUND(100.0 * SUM(Class) / COUNT(*), 3) AS fraud_rate_pct,
        RANK() OVER (ORDER BY 1.0 * SUM(Class) / COUNT(*) DESC) AS rnk
    FROM fraud_transactions
    GROUP BY Hour_of_Day
) ranked
WHERE rnk <= 5;
```

**15. Which loan purposes have an above-average default rate?**

```sql
SELECT purpose, default_rate_pct
FROM (
    SELECT
        purpose,
        ROUND(100.0 * SUM(CASE WHEN credit_risk = 0 THEN 1 ELSE 0 END) / COUNT(*), 2) AS default_rate_pct
    FROM credit_applicants
    GROUP BY purpose
) purpose_summary
WHERE default_rate_pct > (
    SELECT ROUND(100.0 * SUM(CASE WHEN credit_risk = 0 THEN 1 ELSE 0 END) / COUNT(*), 2)
    FROM credit_applicants
);
```

### Very Advanced Queries (CTEs, NTILE, Running Totals, Weighted Scoring)

**16. Flight-risk score: flag customers showing multiple churn warning signs**

```sql
SELECT
    CLIENTNUM,
    Total_Trans_Ct,
    Months_Inactive_12_mon,
    Contacts_Count_12_mon,
    (CASE WHEN Total_Trans_Ct < 45 THEN 1 ELSE 0 END +
     CASE WHEN Months_Inactive_12_mon >= 3 THEN 1 ELSE 0 END +
     CASE WHEN Contacts_Count_12_mon >= 4 THEN 1 ELSE 0 END) AS risk_score
FROM customers
WHERE Attrition_Flag = 'Existing Customer'
ORDER BY risk_score DESC;
```

**17. What is the cumulative fraud value across hours of day?**

```sql
SELECT
    Hour_of_Day,
    ROUND(SUM(CASE WHEN Class = 1 THEN Amount ELSE 0 END), 2) AS fraud_value_this_hour,
    ROUND(SUM(SUM(CASE WHEN Class = 1 THEN Amount ELSE 0 END)) OVER (ORDER BY Hour_of_Day), 2) AS running_fraud_value
FROM fraud_transactions
GROUP BY Hour_of_Day
ORDER BY Hour_of_Day;
```

**18. How do loan applicants split into 4 risk-value quartiles by amount?**

```sql
SELECT
    amount,
    duration,
    Risk_Label,
    NTILE(4) OVER (ORDER BY amount) AS amount_quartile
FROM credit_applicants
ORDER BY amount;
```

**19. Using a CTE, how does churn rate differ across customer spend tiers?**

```sql
WITH tiered_customers AS (
    SELECT
        CLIENTNUM,
        Total_Trans_Amt,
        Churned,
        CASE
            WHEN Total_Trans_Amt < 2500 THEN 'Low Spend'
            WHEN Total_Trans_Amt < 7500 THEN 'Mid Spend'
            ELSE 'High Spend'
        END AS spend_tier
    FROM customers
)
SELECT
    spend_tier,
    COUNT(*) AS customers,
    SUM(Churned) AS churned,
    ROUND(100.0 * SUM(Churned) / COUNT(*), 2) AS churn_rate_pct
FROM tiered_customers
GROUP BY spend_tier
ORDER BY churn_rate_pct DESC;
```

**20. Which high-fee card tiers have above-average churn? (Join + derived subquery)**

```sql
SELECT d.Card_Category, d.Annual_Fee, churn_summary.churn_rate_pct
FROM card_tier_dim d
JOIN (
    SELECT Card_Category, ROUND(100.0*SUM(Churned)/COUNT(*),2) AS churn_rate_pct
    FROM customers GROUP BY Card_Category
) churn_summary ON churn_summary.Card_Category = d.Card_Category
WHERE d.Annual_Fee > 0
  AND churn_summary.churn_rate_pct > (SELECT ROUND(100.0*SUM(Churned)/COUNT(*),2) FROM customers)
ORDER BY churn_summary.churn_rate_pct DESC;
```

---

## Tools & Technologies

| Tool | Purpose |
|---|---|
| **Python (pandas)** | Data cleaning, feature engineering, dimension table creation, integrity validation |
| **SQLite / SQL** | Grouping, `CASE` logic, simple joins, window functions (`RANK`, `PERCENT_RANK`, `NTILE`), correlated subqueries, CTEs |
| **Power BI** | Interactive dashboard, data visualization, slicers |
| **DAX** | `CALCULATE`, `DIVIDE`, `ALL`, measures for KPIs and risk indexing |

---

## Key Insights

- **Overall churn rate: 16.07%** (1,627 of 10,127 customers)
- **Low-spend customers churn at nearly 2x the overall rate** — 30.78% vs. 16.07% overall, the sharpest single risk signal in the dataset
- **Churned customers show clear declining engagement beforehand** — 44.9 avg transactions vs. 68.7 for retained customers, confirming transaction activity as a leading indicator
- **2,754 customers (27.19% of the base) qualify as high-value** (credit limit > $10K), concentrated in Gold/Platinum tiers
- **Fraud is rare but costly** — only 0.167% of transactions (473 of 283,726) are fraudulent, yet fraud transactions average $123.87 vs. $88.41 for legitimate ones, a 40% gap
- **Fraud risk is 4x higher overnight** — the Night shift (12AM–6AM) has a 0.482% fraud rate vs. 0.115% in the Evening, and hour 2 AM alone hits 1.451% — over 8x the overall average
- **30% of loan applicants are high-risk**, and these accounts hold **36.12% of total loan value** — risk is disproportionately concentrated in loan value, not just applicant count
- **Education-related loans default most** (41.67% by category; "retraining" alone hits 44% individually), while Business loans default least (11.11%)
- **Platinum cardholders churn the most despite the highest annual fee** — 25% churn rate vs. 14.77% for Silver, suggesting premium customers may not feel the fee is justified by their experience

---

## Dashboard

### Executive Summary

> **Headline Stat**
> $1.18M in High-Risk Loan Exposure Identified
> Analyzed 294,853 records across customers, transactions, and loan applicants to uncover where churn, fraud, and credit risk concentrate — and what to act on first.

> **Key Findings**
> Low-spend customers churn at nearly 2x the overall rate. 2,754 customers (27%) qualify as high-value and warrant retention priority. Fraud is rare (0.17%) but skews toward higher-value transactions, peaking sharply overnight. 30% of loan applicants are high-risk, holding a disproportionate 36% of total loan value.

> **Recommendations**
> Launch a proactive retention program targeting customers with declining transaction activity, before formal churn.
> Prioritize relationship management on the 2,754 identified high-value customers.
> Tighten fraud monitoring specifically around the overnight risk window and higher-value transactions.
> Reassess underwriting thresholds for loan purposes most linked to the 36% high-risk loan value concentration.

![Executive Summary](dashboard_images/executive_summary.png)

### Customer Churn & Retention
![Churn & Retention Dashboard](dashboard_images/churn_retention.png)

### Customer Segmentation
![Segmentation Dashboard](dashboard_images/customer_segmentation.png)

### Transaction Fraud Detection
![Fraud Detection Dashboard](dashboard_images/fraud_detection.png)

### Credit Risk & Loan Default
![Credit Risk Dashboard](dashboard_images/credit_risk.png)

**Live Dashboard:** *[ADD LIVE DASHBOARD LINK HERE]*

---

## Results & Conclusion

This project shows that risk and value in this data are not evenly spread — they concentrate in identifiable segments. Low-spend customers drive a disproportionate share of churn, a small fraction of high-value customers represent outsized relationship value, fraud clusters sharply around a narrow overnight window, and loan risk concentrates by both purpose and amount rather than spreading evenly across the portfolio. The dashboard suite translates these findings into an actionable tool: rather than treating churn, fraud, and credit risk as separate reactive problems, a bank can use these concentrated signals to intervene early and allocate resources where they matter most.

**Recommended next steps for the business:**
1. Build a proactive retention workflow for low-spend and declining-engagement customers, the segment most likely to churn.
2. Extend fraud monitoring rules specifically around the overnight window and higher-value transactions, rather than applying uniform scrutiny across all hours.
3. Revisit underwriting criteria for loan purposes with above-average default rates (e.g. retraining, education), since risk is concentrated rather than evenly distributed.

---

