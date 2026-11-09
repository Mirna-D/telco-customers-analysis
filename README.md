# Telco Customer Churn Analytics & Dashboard Portfolio

[![Platform: Google BigQuery](https://img.shields.io/badge/Platform-Google%20BigQuery-orange?logo=googlecloud)](https://cloud.google.com/bigquery)
[![Tool: Power BI](https://img.shields.io/badge/Visualization-Power%20BI-yellow?logo=powerbi)](https://powerbi.microsoft.com/)
[![Author](https://img.shields.io/badge/Author-Mirnawati%20Darwis-blue)](https://github.com/)

An end-to-end data analytics project examining customer attrition drivers, demographic risk profiles, and service-level vulnerabilities for a telecommunications company. This portfolio demonstrates data preparation, cloud data warehousing in Google BigQuery, and executive-level interactive dashboarding in Power BI.

---

## 📋 Table of Contents
1. [Project Overview](#-project-overview)
2. [Dataset & Schema](#-dataset--schema)
3. [Data Preparation & Quality Audit (BigQuery)](#-data-preparation--quality-audit-bigquery)
4. [SQL Analytics & Insights Queries](#-sql-analytics--insights-queries)
5. [Key Findings & Business Insights](#-key-findings--business-insights)
6. [Actionable Strategic Recommendations](#-actionable-strategic-recommendations)
7. [Project Structure](#-project-structure)

---

## 🎯 Project Overview
Customer attrition (churn) is one of the most critical metrics affecting recurring revenue in the telecommunications industry. This project analyzes **7,043 customer records** to uncover root causes of churn, evaluate high-risk customer segments, and quantify potential monthly recurring revenue (MRR) at risk. 

The analytical workflow spans two primary dashboard pages:
* **Page 1: Executive & Customer Profile** – Focuses on executive summary KPIs, tenure cohort drop-off trends, contract risk analysis, and demographic vulnerabilities.
* **Page 2: Service & Financial Drivers** – Examines payment channels, internet service tiers, security/support bundle protection matrices, and streaming engagement patterns.

---

## 🗄️ Dataset & Schema
* **Source:** Kaggle Telco Customer Churn Dataset
* **Database Host:** Google BigQuery (`telco-customer-churn-513509.telco`)
* **Tables:**
  * Raw Data: `telco-customer-churn-513509.telco.telco_customer_churn`
  * Cleaned Dataset: `telco-customer-churn-513509.telco.telco_cleaned_data`

### Core Schema Fields:
* `customerID`: Unique customer identifier (STRING)
* `gender`: Customer gender (`Female`, `Male`)
* `SeniorCitizen`: Senior citizen indicator (`0`, `1`)
* `Partner`: Whether the customer has a partner (`BOOLEAN`)
* `Dependents`: Whether the customer has dependents (`BOOLEAN`)
* `tenure`: Number of months the customer has stayed with the company (`INTEGER`)
* `Contract`: Subscription contract type (`Month-to-month`, `One year`, `Two year`)
* `PaymentMethod`: Customer payment channel (`Electronic check`, `Mailed check`, `Bank transfer (automatic)`, `Credit card (automatic)`)
* `InternetService`: Type of internet service (`DSL`, `Fiber optic`, `No`)
* `MonthlyCharges`: Amount charged to the customer monthly (`FLOAT`)
* `TotalCharges`: Total amount charged to the customer (`FLOAT`)
* `TenureGroup`: Derived cohort grouping (`0-12 Months`, `13-24 Months`, `25-48 Months`, `>48 Months`)
* `Churn`: Whether the customer churned (`BOOLEAN`)

---

## 🛠️ Data Preparation & Quality Audit (BigQuery)
Before analytics queries were executed, a rigorous data quality audit and transformation pipeline was executed in BigQuery to ensure data integrity:
1. **Primary Key & Duplicate Audit:** Checked for duplicate `customerID` records.
2. **Missing Value Audit:** Inspected nulls and blank spaces (`' '`) in numerical attributes such as `TotalCharges` (handling new accounts where tenure = 0).
3. **Data Cleaning & Cohort Derivation:** Built a cleaned dataset table retaining original case formatting while introducing `TenureGroup` for cohort analysis.

```sql
-- Cleaned Dataset Transformation Table in BigQuery
CREATE OR REPLACE TABLE `telco-customer-churn-513509.telco.telco_cleaned_data` AS
WITH DeduplicatedData AS (
    SELECT *,
           ROW_NUMBER() OVER(PARTITION BY TRIM(customerID) ORDER BY tenure DESC) AS row_num
    FROM `telco-customer-churn-513509.telco.telco_customer_churn`
    WHERE customerID IS NOT NULL AND TRIM(customerID) != ''
)
SELECT
    TRIM(customerID) AS customerID,
    gender,
    SeniorCitizen,
    Partner,
    Dependents,
    SAFE_CAST(tenure AS INT64) AS tenure,
    PhoneService,
    MultipleLines,
    InternetService,
    OnlineSecurity,
    OnlineBackup,
    DeviceProtection,
    TechSupport,
    StreamingTV,
    StreamingMovies,
    Contract,
    PaperlessBilling,
    PaymentMethod,
    SAFE_CAST(MonthlyCharges AS FLOAT64) AS MonthlyCharges,
    COALESCE(
        SAFE_CAST(TotalCharges AS FLOAT64), 
        SAFE_CAST(MonthlyCharges AS FLOAT64) * SAFE_CAST(tenure AS INT64), 
        0.0
    ) AS TotalCharges,
    CASE 
        WHEN SAFE_CAST(tenure AS INT64) <= 12 THEN '0-12 Months'
        WHEN SAFE_CAST(tenure AS INT64) BETWEEN 13 AND 24 THEN '13-24 Months'
        WHEN SAFE_CAST(tenure AS INT64) BETWEEN 25 AND 48 THEN '25-48 Months'
        ELSE '>48 Months'
    END AS TenureGroup,
    Churn
FROM DeduplicatedData
WHERE row_num = 1;
```

---

## 📊 SQL Analytics & Insights Queries
The complete analytical script (`telco_churn_queries.sql`) covering all dashboard metrics can be executed in BigQuery:

```sql
-- Executive Summary KPIs Query
SELECT 
    COUNT(customerID) AS total_customers,
    COUNTIF(Churn = 'Yes') AS total_churn,
    ROUND(COUNTIF(Churn = 'Yes') * 100.0 / COUNT(customerID), 1) AS churn_rate_pct,
    ROUND(SUM(CASE WHEN Churn = 'Yes' THEN MonthlyCharges ELSE 0 END), 2) AS monthly_revenue_lost
FROM `telco-customer-churn-513509.telco.telco_cleaned_data`;
```

---

## 💡 Key Findings & Business Insights
1. **0-12 Month Critical Drop-off Window:** Nearly **50% of overall churned users** cancel their service within the first 12 months (`0-12 Months` cohort exhibits a **47% churn rate**).
2. **Month-to-Contract Vulnerability:** Month-to-month contracts exhibit a severe 42.7% churn rate, compared to just 2.8% for 2-year terms. There are **2,220 active month-to-month customers** representing **$136.45K/mo** in active MRR at risk.
3. **Senior Single Demographic Risk:** Demographic analysis reveals that **Senior Singles experience the highest churn rate at 49%**, whereas Married Non-Seniors represent the most loyal customer tier (17% churn).
4. **Payment Channel Correlation:** **Electronic check** payment methods trigger the highest churn rate at **45%**, heavily outperforming automated payment channels.
5. **Security and Support Protection:** Customers lacking both Tech Support and Online Security experience extreme churn rates up to **49%**, proving that security add-ons act as powerful retention buffers.

---

## 🚀 Actionable Strategic Recommendations
* **Contract Migration Incentives:** Provide monthly discount incentives for month-to-month subscribers who transition to 1-year or 2-year plans to secure long-term retention.
* **Proactive Early Onboarding:** Deploy automated check-in workflows, introductory support guides, and loyalty perks during the critical first 90 days of subscription.
* **Payment Channel Migration Campaign:** Offer fee waivers or billing credits to incentivize customers to migrate from Electronic checks to automated payment systems (Auto-Pay).
* **Security Bundle Integration:** Shift promotional focus from streaming add-ons to mandatory Tech Support and Online Security bundles for Fiber optic users to cut down extreme churn risks.

---

## 📂 Project Structure
```text
├── README.md               # Project documentation
├── telco_churn_queries.sql # Complete BigQuery data prep & analytics queries
└── assets/                 # Dashboard exports / screenshots
```

---
*© 2026 Telco Churn Analysis | **Mirnawati Darwis** (BigQuery & Power BI)*