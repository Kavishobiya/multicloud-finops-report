# Enterprise Multi-Cloud FinOps Engine (AWS • Azure • GCP)

![FOCUS Specification](https://img.shields.io/badge/Schema-FOCUS%201.0%20%2F%201.2-blue?style=flat-square)
![AWS Athena](https://img.shields.io/badge/SQL%20Engine-AWS%20Athena-orange?style=flat-square)
![Power BI](https://img.shields.io/badge/BI-Power%20BI-yellow?style=flat-square)
![Excel](https://img.shields.io/badge/Financial%20Modeling-Excel-darkgreen?style=flat-square)
![Cloud Spend](https://img.shields.io/badge/Cloud%20Spend-%241.32M%20Audited-green?style=flat-square)

An enterprise-grade Cloud Financial Management (FinOps) pipeline that unifies, normalizes, and governs multi-cloud billing datasets across **AWS**, **Microsoft Azure**, and **Google Cloud Platform (GCP)** using the **FinOps Open Cost and Usage Specification (FOCUS)**.

This repository demonstrates the complete FinOps lifecycle: from serverless data lake ingestion and schema parsing in **AWS Athena (SQL)**, to unit economics and showback modeling in **Power BI**, and commitment break-even analysis (Savings Plans & Reserved Instances) in **Microsoft Excel**.

---

## 🏛️ Architecture Overview

```
                                      [ Multi-Cloud Ingestion ]
                                                 │
          ┌──────────────────────────────────────┼──────────────────────────────────────┐
          ▼                                      ▼                                      ▼
    AWS Billing Feed                      Azure Cost Export                      GCP Cloud Billing
  (S3 / FOCUS 1.0/1.2)                 (Blob / FOCUS 1.0/1.2)                 (Bucket / FOCUS 1.0/1.2)
          │                                      │                                      │
          └──────────────────────────────────────┬──────────────────────────────────────┘
                                                 ▼
                             [ Amazon S3 Central Cost Data Lake ]
                                `s3://finops-lake/focus-data/`
                                                 │
                                                 ▼
                              [ Serverless Query Engine: Athena ]
                                  - Hive OpenCSVSerDe
                                  - Dynamic JSON Tag Extraction
                                  - Unified Curated View (`v_focus_curated`)
                                                 │
                        ┌────────────────────────┴────────────────────────┐
                        ▼                                                 ▼
             [ Power BI Dashboard ]                          [ Excel Financial Modeling ]
        - Executive Multi-Cloud Split                   - 1-Yr vs. 3-Yr Commitment Scenarios
        - Cross-Cloud Department Showback               - Effective Savings Rate (ESR) Targets
        - Automated Run-Rate Forecasting                - Break-even Utilization Matrices
        - Tag Compliance & Hygiene Audits
```

---

## 📊 Dataset Profile & Financial Scope

* **Audited Period:** Rolling 6-Month Window (`2026-05-01` to `2026-10-31`)
* **Total Audited Spend:** **$1,321,656.23 Effective Cost** ($1,234,233.23 Billed Cost) across 13,496 billing line items
* **Provider Distribution:**
  * **AWS (Core Platform & Microservices):** $590,230.28 (44.66%) | Monthly Target: $90k
  * **Azure (Enterprise ERP & Hybrid IT):** $410,652.15 (31.07%) | Monthly Target: $65k
  * **GCP (BigQuery Analytics & AI/ML):** $320,773.80 (24.27%) | Monthly Target: $50k
* **Key Workload Categories:** Compute ($553.8K), Database ($204.7K), Networking ($185.0K), Management ($131.1K), Storage ($123.8K), Analytics ($123.4K)

---

## 💡 Key FinOps Problems Solved

### 1. Inconsistent Multi-Cloud Tagging & Casing Discrepancies
* **Problem:** In multi-cloud environments, resource tagging is fragmented. AWS resources often use PascalCase (`{"CostCenter": "Engineering"}`), Azure uses lowercase (`{"costcenter": "Engineering"}`), and GCP instances often omit tags entirely, causing showback queries to drop valid spend into untagged buckets.
* **Solution:** Created an Athena curated view (`v_multicloud_curated`) that parses JSON tags with fallback cascades using `COALESCE`, `NULLIF`, and `json_extract_scalar`, recovering and attributing departmental spend automatically.

### 2. Upfront vs. Amortized Discrepancy (Cash vs. Accrual)
* **Problem:** Finance teams require accrual accounting for chargeback, whereas raw invoices (`BilledCost`) register massive one-off cash spikes when 1-Year or 3-Year Reservations are purchased upfront.
* **Solution:** Modeled dual-metric reporting separating `BilledCost` (cash flow view) from `EffectiveCost` (amortized consumption rate), preventing artificial department budget variance during commitment renewal cycles.

### 3. Anomaly Detection & Incident Remediation
Successfully flagged and isolated three deterministic multi-cloud cost anomalies:
* **AWS Egress Spike (July 12–16, 2026):** Runaway S3 cross-region replication and NAT Gateway transfer peaking at **$10,534.43/day** (vs. baseline ~$3,000/day).
* **Azure GPU VM Leak (Sept 19–20, 2026):** High-tier HPC virtual machines left running unmanaged over a 48-hour weekend, driving a **+$5,200** unbudgeted spike.
* **GCP BigQuery Runaway Scan (August 21, 2026):** Unpartitioned ad-hoc analytics scan processing 820 TB, causing a **+$4,100** single-day anomaly.

---

## 🛠️ Repository File Structure

```text
├── README.md
|
|
├── dashboards/
│   ├── multicloud_finops_report.pbix      # Power BI Interactive Report
│   └── screenshots/                       # High-resolution dashboard previews
│       ├── executive_overview.png
│       ├── department_showback.png
│       └── tag_governance.png
└── data/
    └── focus_multicloud_cost_and_usage.csv
```

---

## 🚀 Step-by-Step Implementation

### Step 1: Ingestion & DDL Setup in AWS Athena
Using `OpenCSVSerde` ensures double-quoted JSON strings inside the `Tags` column are not corrupted by internal commas:

```sql
CREATE DATABASE IF NOT EXISTS finops_db;

CREATE EXTERNAL TABLE IF NOT EXISTS finops_db.multicloud_costs (
  BillingPeriodStart STRING,
  BillingPeriodEnd STRING,
  ChargePeriodStart STRING,
  ChargePeriodEnd STRING,
  ProviderName STRING,
  BillingAccountId STRING,
  BillingAccountName STRING,
  SubAccountId STRING,
  SubAccountName STRING,
  RegionId STRING,
  RegionName STRING,
  ServiceName STRING,
  ServiceCategory STRING,
  PricingCategory STRING,
  ChargeCategory STRING,
  ChargeSubcategory STRING,
  PricingUnit STRING,
  UsageQuantity DOUBLE,
  ListCost DOUBLE,
  ContractedCost DOUBLE,
  BilledCost DOUBLE,
  EffectiveCost DOUBLE,
  BillingCurrency STRING,
  Tags STRING,
  x_AllocationType STRING,
  x_TagCompliance STRING
)
ROW FORMAT SERDE 'org.apache.hadoop.hive.serde2.OpenCSVSerde'
WITH SERDEPROPERTIES (
  'separatorChar' = ',',
  'quoteChar'     = '"',
  'escapeChar'    = '\\'
)
STORED AS TEXTFILE
LOCATION 's3://YOUR-BUCKET-NAME/focus-data/'
TBLPROPERTIES ("skip.header.line.count"="1");
```

---

### Step 2: Normalization & Defensive Tag Extraction
To protect downstream BI models from missing or case-variant tag keys, the view extracts values dynamically:

```sql
CREATE OR REPLACE VIEW finops_db.v_multicloud_curated AS
SELECT 
    TRIM(ProviderName) AS ProviderName,
    CAST(SUBSTRING(ChargePeriodStart, 1, 10) AS DATE) AS UsageDate,
    SUBSTRING(ChargePeriodStart, 1, 7) AS BillingMonth,
    TRIM(SubAccountId) AS SubAccountId,
    TRIM(SubAccountName) AS SubAccountName,
    TRIM(ServiceName) AS ServiceName,
    TRIM(ServiceCategory) AS ServiceCategory,
    TRIM(PricingCategory) AS PricingCategory,
    TRIM(ChargeCategory) AS ChargeCategory,
    BilledCost,
    EffectiveCost,
    ListCost,
    -- Case-insensitive JSON Tag Fallback Hierarchy
    COALESCE(
        NULLIF(json_extract_scalar(Tags, '$.CostCenter'), ''),
        NULLIF(json_extract_scalar(Tags, '$.costcenter'), ''),
        'Unallocated'
    ) AS CostCenter,
    COALESCE(
        NULLIF(json_extract_scalar(Tags, '$.Environment'), ''),
        NULLIF(json_extract_scalar(Tags, '$.environment'), ''),
        'Untagged'
    ) AS Environment,
    x_AllocationType,
    x_TagCompliance
FROM finops_db.multicloud_costs;
```

---

### Step 3: Core Analytical Queries

#### 1. Cross-Cloud Departmental Showback Matrix
```sql
SELECT 
    CostCenter,
    ProviderName,
    ROUND(SUM(EffectiveCost), 2) AS AmortizedSpend,
    ROUND(SUM(BilledCost), 2) AS InvoicedSpend
FROM finops_db.v_multicloud_curated
GROUP BY CostCenter, ProviderName
ORDER BY AmortizedSpend DESC;
```

#### 2. Tag Governance & Hygiene Compliance
```sql
SELECT 
    ProviderName,
    x_TagCompliance,
    COUNT(*) AS LineItemCount,
    ROUND(SUM(EffectiveCost), 2) AS SpendVolume,
    ROUND((SUM(EffectiveCost) / (SELECT SUM(EffectiveCost) FROM finops_db.v_multicloud_curated)) * 100, 2) AS SpendSharePct
FROM finops_db.v_multicloud_curated
GROUP BY ProviderName, x_TagCompliance
ORDER BY ProviderName, SpendVolume DESC;
```

---

## 📈 Power BI Reporting Suite

The accompanying Power BI workbook (`multicloud_finops_report.pbix`) includes three core operational pages:

1. **Executive Multi-Cloud Overview:**
   * **KPIs:** Month-to-Date (MTD) Spend, Run-Rate Forecast, Effective Savings Rate (ESR).
   * **Visuals:** Provider spend share donut chart, historical month-over-month trend line, cross-cloud service category Pareto chart.
2. **Department Showback & Unit Economics:**
   * **Matrix Table:** Hierarchical drill-down (`CostCenter -> Environment -> ServiceName`).
   * **Slicers:** Slicer dropdowns for `ProviderName`, `SubAccountName`, and `BillingMonth`.
3. **FinOps Governance & Anomaly Detection:**
   * **Visuals:** Tag hygiene compliance tracker (`Full` vs. `Partial` vs. `Untagged`), anomaly spike flags, and idle compute identification.

### Key DAX Measures
```dax
// Projected Month-End Run Rate
Projected Run Rate = 
VAR CurrentSpend = SUM(v_multicloud_curated[EffectiveCost])
VAR DaysElapsed = DAY(MAX(v_multicloud_curated[UsageDate]))
VAR TotalDaysInMonth = DAY(EOMONTH(MAX(v_multicloud_curated[UsageDate]), 0))
RETURN 
DIVIDE(CurrentSpend, DaysElapsed, 0) * TotalDaysInMonth
```

```dax
// Tag Compliance Percentage
Tag Compliance % = 
DIVIDE(
    CALCULATE(SUM(v_multicloud_curated[EffectiveCost]), v_multicloud_curated[CostCenter] <> "Unallocated"),
    SUM(v_multicloud_curated[EffectiveCost]),
    0
) * 100
```

---

## 💼 Excel Commitment Modeling (Savings Plans / RIs)

The Excel financial model evaluates commitment strategies across Compute & Database services:
* **Current On-Demand Compute Baseline:** $60,000 / month
* **Target Coverage Level:** 75%
* **1-Year Commitment (No Upfront):** 28% discount $
ightarrow$ Projected Annual Savings: **$151,200**
* **3-Year Commitment (Partial Upfront):** 48% discount $
ightarrow$ Projected Annual Savings: **$259,200**
* **Break-Even Analysis:** Identifies the minimum compute run-rate utilization threshold (62% for 1-Yr, 54% for 3-Yr) required to neutralize commitment risk.

---

## 👤 Kavishobiya

**Cloud FinOps Practitioner**  
* Certification: FinOps Certified Practitioner (FOCP)  
* Core Competencies: Cloud Financial Management, FOCUS 1.0/1.2, SQL (Athena/Presto), Power BI, Financial Modeling, AWS CUR & Azure Cost Management.  
* Actively seeking opportunities in Cloud FinOps Analysis, Cost Optimization, and Cloud Financial Engineering.
