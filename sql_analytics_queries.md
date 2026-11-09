-- ============================================================================
-- TELCO CUSTOMER CHURN: COMPLETE SQL ANALYTICS & DASHBOARD QUERIES
-- Database Dialect: Google BigQuery
-- Project/Dataset: telco-customer-churn-513509.telco
-- Tables: `telco_customer_churn` (Raw), `telco_cleaned_data` (Cleaned)
-- Author: Mirnawati Darwis
-- ============================================================================

-------------------------------------------------------------------------------
-- STEP 0: DATA PREPARATION, QUALITY AUDIT & CLEANING
-------------------------------------------------------------------------------

-- 0.1 Duplicate Check on Primary Key (customerID)
SELECT 
    customerID, 
    COUNT(*) AS record_count
FROM `telco-customer-churn-513509.telco.telco_customer_churn`
GROUP BY customerID
HAVING COUNT(*) > 1;

-- 0.2 Missing Value & Null Audit
SELECT
    COUNTIF(customerID IS NULL OR TRIM(customerID) = '') AS null_customer_id,
    COUNTIF(tenure IS NULL) AS null_tenure,
    COUNTIF(MonthlyCharges IS NULL) AS null_monthly_charges,
    COUNTIF(TotalCharges IS NULL OR SAFE_CAST(TotalCharges AS STRING) = ' ' OR TRIM(SAFE_CAST(TotalCharges AS STRING) = '') AS missing_total_charges,
    COUNTIF(Churn IS NULL) AS null_churn
FROM `telco-customer-churn-513509.telco.telco_customer_churn`;

-- 0.3 Build Cleaned Dataset Table (`telco_cleaned_data`)
-- Retaining original case values and adding TenureGroup grouping
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
    -- Added TenureGroup cohort classification as requested
    CASE 
        WHEN SAFE_CAST(tenure AS INT64) <= 12 THEN '0-12 Months'
        WHEN SAFE_CAST(tenure AS INT64) BETWEEN 13 AND 24 THEN '13-24 Months'
        WHEN SAFE_CAST(tenure AS INT64) BETWEEN 25 AND 48 THEN '25-48 Months'
        ELSE '>48 Months'
    END AS TenureGroup,
    Churn
FROM DeduplicatedData
WHERE row_num = 1;


-------------------------------------------------------------------------------
-- PAGE 1: EXECUTIVE & CUSTOMER PROFILE QUERIES
-------------------------------------------------------------------------------

-- 1.1 Executive Summary KPIs (Total Customers, Total Churn, Churn Rate %, Revenue Lost)
SELECT 
    COUNT(customerID) AS total_customers,
    COUNTIF(Churn = 'Yes') AS total_churn,
    ROUND(COUNTIF(Churn = 'Yes') * 100.0 / COUNT(customerID), 1) AS churn_rate_pct,
    ROUND(SUM(CASE WHEN Churn = 'Yes' THEN MonthlyCharges ELSE 0 END), 2) AS monthly_revenue_lost
FROM `telco-customer-churn-513509.telco.telco_cleaned_data`;

-- 1.2 Churn Rate by Tenure Cohorts (Line Chart)
SELECT 
    TenureGroup,
    COUNT(customerID) AS total_customers,
    COUNTIF(Churn = 'Yes') AS churned_customers,
    ROUND(COUNTIF(Churn = 'Yes') * 100.0 / COUNT(customerID), 1) AS churn_rate_pct
FROM `telco-customer-churn-513509.telco.telco_cleaned_data`
GROUP BY TenureGroup
ORDER BY 
    CASE TenureGroup
        WHEN '0-12 Months' THEN 1
        WHEN '13-24 Months' THEN 2
        WHEN '25-48 Months' THEN 3
        WHEN '>48 Months' THEN 4
    END;

-- 1.3 Churn Rate by Contract Type (Bar Chart)
SELECT 
    Contract,
    COUNT(customerID) AS total_customers,
    COUNTIF(Churn = 'Yes') AS churned_customers,
    ROUND(COUNTIF(Churn = 'Yes') * 100.0 / COUNT(customerID), 1) AS churn_rate_pct
FROM `telco-customer-churn-513509.telco.telco_cleaned_data`
GROUP BY Contract
ORDER BY churn_rate_pct DESC;

-- 1.4 Demographic Churn (Senior Status & Relationship Status)
SELECT 
    CASE WHEN SeniorCitizen = '1' THEN 'Senior Citizen' ELSE 'Non-Senior' END AS senior_status,
    CASE WHEN Partner = 'Yes' THEN 'Married' ELSE 'Single' END AS relationship_status,
    COUNT(customerID) AS total_customers,
    COUNTIF(Churn = 'Yes') AS churned_customers,
    ROUND(COUNTIF(Churn = 'Yes') * 100.0 / COUNT(customerID), 1) AS churn_rate_pct
FROM `telco-customer-churn-513509.telco.telco_cleaned_data`
GROUP BY SeniorCitizen, Partner
ORDER BY churn_rate_pct DESC;


-------------------------------------------------------------------------------
-- PAGE 2: SERVICE & FINANCIAL DRIVERS QUERIES
-------------------------------------------------------------------------------

-- 2.1 Service & Financial KPIs (High-Risk Month-to-Month Active, Avg Charges, MRR at Risk)
SELECT 
    COUNTIF(Contract = 'Month-to-month' AND Churn = 'No') AS month_to_month_active,
    ROUND(AVG(CASE WHEN Churn = 'Yes' THEN MonthlyCharges END), 2) AS avg_charge_churners,
    ROUND(AVG(CASE WHEN Churn = 'No' THEN MonthlyCharges END), 2) AS avg_charge_loyal,
    ROUND(SUM(CASE WHEN Contract = 'Month-to-month' AND Churn = 'No' THEN MonthlyCharges END), 2) AS active_mrr_at_risk
FROM `telco-customer-churn-513509.telco.telco_cleaned_data`;

-- 2.2 Churn Rate by Payment Method
SELECT 
    PaymentMethod,
    COUNT(customerID) AS total_customers,
    COUNTIF(Churn = 'Yes') AS churned_customers,
    ROUND(COUNTIF(Churn = 'Yes') * 100.0 / COUNT(customerID), 1) AS churn_rate_pct
FROM `telco-customer-churn-513509.telco.telco_cleaned_data`
GROUP BY PaymentMethod
ORDER BY churn_rate_pct DESC;

-- 2.3 Churn Rate by Internet Service Type
SELECT 
    InternetService,
    COUNT(customerID) AS total_customers,
    COUNTIF(Churn = 'Yes') AS churned_customers,
    ROUND(COUNTIF(Churn = 'Yes') * 100.0 / COUNT(customerID), 1) AS churn_rate_pct
FROM `telco-customer-churn-513509.telco.telco_cleaned_data`
GROUP BY InternetService
ORDER BY churn_rate_pct DESC;

-- 2.4 Matrix Heatmap: TechSupport vs OnlineSecurity Churn Rate
SELECT 
    CASE WHEN TechSupport = 'Yes' THEN 'With Tech Support' ELSE 'No Tech Support' END AS tech_support_status,
    CASE WHEN OnlineSecurity = 'Yes' THEN 'With Security' ELSE 'No Security' END AS security_status,
    COUNT(customerID) AS total_customers,
    COUNTIF(Churn = 'Yes') AS churned_customers,
    ROUND(COUNTIF(Churn = 'Yes') * 100.0 / COUNT(customerID), 1) AS churn_rate_pct
FROM `telco-customer-churn-513509.telco.telco_cleaned_data`
GROUP BY tech_support_status, security_status
ORDER BY churn_rate_pct DESC;

-- 2.5 Treemap: Count of Customers by Streaming Service (StreamingTV and StreamingMovies)
SELECT 
    CONCAT(StreamingTV, ' ', StreamingMovies) AS streaming_combination,
    COUNT(customerID) AS total_customers,
    ROUND(COUNT(customerID) / 1000.0, 2) AS total_customers_in_k
FROM `telco-customer-churn-513509.telco.telco_cleaned_data`
GROUP BY streaming_combination
ORDER BY total_customers DESC;