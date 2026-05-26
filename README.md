# 🏦 NBI Banking Data Pipeline — Azure Data Factory Capstone

![Azure](https://img.shields.io/badge/Azure-Data%20Factory-0078D4?style=for-the-badge&logo=microsoft-azure&logoColor=white)
![SQL](https://img.shields.io/badge/Azure-SQL%20Database-CC2927?style=for-the-badge&logo=microsoft-sql-server&logoColor=white)
![Blob](https://img.shields.io/badge/Azure-Blob%20Storage-0089D6?style=for-the-badge&logo=microsoft-azure&logoColor=white)
![KeyVault](https://img.shields.io/badge/Azure-Key%20Vault-FFD700?style=for-the-badge&logo=microsoft-azure&logoColor=black)
![Status](https://img.shields.io/badge/Status-Completed-2EA44F?style=for-the-badge)

> **National Bank of India (NBI)** — End-to-end Azure Data Factory ETL pipeline solution for banking data ingestion, transformation, and warehousing.

---

## 📋 Project Overview

| Field | Details |
|-------|---------|
| **Assignment** | ADF Capstone — Banking Data Pipeline |
| **Student** | Rudra V. Gandhi |
| **University** | Navrachana University (NUV) |
| **Duration** | 5 Working Days |
| **ADF Instance** | `adf-nbi-banking` |
| **SQL Database** | `sqldb-nbi-dw` |
| **Total Tables** | 12 (5 Dim + 4 Fact + 3 Agg) |
| **Total Records** | 12,507+ rows across all tables |

---

## 🏗️ Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                    SOURCE LAYER (Azure Blob Storage)             │
│  bank_customers.csv  │  bank_accounts.csv  │  bank_employees.csv │
│  bank_transactions.json  │  bank_loans.json  │  bank_fraud_alerts.csv │
└─────────────────────────────┬───────────────────────────────────┘
                              │
                    ┌─────────▼─────────┐
                    │  Azure Data Factory │
                    │   adf-nbi-banking  │
                    │                   │
                    │  ┌─────────────┐  │
                    │  │ Linked Svcs │  │
                    │  │  Datasets   │  │
                    │  │  Data Flows │  │
                    │  │  Pipelines  │  │
                    │  └─────────────┘  │
                    └─────────┬─────────┘
                              │
                    ┌─────────▼─────────┐
                    │  Azure SQL Database│
                    │   sqldb-nbi-dw    │
                    │  12 Target Tables  │
                    └───────────────────┘
```

---

## 📁 Repository Structure

```
NBI-ADF-Banking-Project/
│
├── 📂 arm-template/                  # ADF ARM Template Export
│   ├── ARMTemplateForFactory.json
│   └── ARMTemplateParametersForFactory.json
│
├── 📂 sql-scripts/                   # SQL Scripts
│   └── NBI_ADF_Target_Tables.sql     # Creates all 12 target tables
│
├── 📂 source-data/                   # Sample Source Files
│   ├── bank_customers.csv
│   ├── bank_accounts.csv
│   ├── bank_employees.csv
│   ├── bank_fraud_alerts.csv
│   ├── bank_transactions.json
│   └── bank_loans.json
│
├── 📂 screenshots/                   # Pipeline Run Screenshots
│   ├── pl_load_dimensions.png
│   ├── pl_load_fact_transactions.png
│   ├── pl_load_fact_loans.png
│   ├── pl_load_fraud_and_balances.png
│   ├── pl_build_aggregations.png
│   ├── pl_master_bank_etl.png
│   ├── ROW_Count_1.png
│   └── ROW_Count_2.png
│
├── 📂 docs/                          # Documentation
│   └── NBI_ADF_Complete_Technical_Document.docx
│
└── README.md                         # This file
```

---

## ☁️ Azure Resources

| Resource | Name | Purpose |
|----------|------|---------|
| Resource Group | `rg-nbi-banking` | Container for all resources |
| Azure Data Factory | `adf-nbi-banking` | ETL orchestration |
| Azure Blob Storage | `stgnbibanking` | Source file storage |
| Azure SQL Database | `sqldb-nbi-dw` | Target data warehouse |
| Azure Key Vault | `kv-nbi-banking` | Secret management |
| Log Analytics | `law-nbi-banking` | Diagnostic logging |

---

## 🔗 Linked Services & Datasets

### Linked Services (2)
| Name | Type | Auth |
|------|------|------|
| `ls_blob_source` | Azure Blob Storage | Account Key |
| `ls_sql_dw` | Azure SQL Database | SQL Authentication |

### Datasets (18)
| Folder | Count | Type |
|--------|-------|------|
| Blob_csv | 4 | CSV (Customer, Account, Employee, Fraud) |
| Blob_json | 2 | JSON (Loans, Transactions) |
| SQL_Dataset | 12 | Azure SQL (all target tables) |

---

## 🔄 Pipeline Design

### Master Pipeline Flow
```
pl_load_dimensions
        ↓ (Success)
pl_load_fact_transactions
        ↓ (Success)
pl_load_fact_loans ──────────────┐
                                  ├── pl_build_aggregations
pl_load_fraud_and_balances ──────┘
```

### 6 Pipelines

#### 1️⃣ `pl_load_dimensions`
Loads 5 dimension tables **sequentially**.

| Data Flow | Source | Target | Key Logic |
|-----------|--------|--------|-----------|
| `dfdimcustomers` | bank_customers.csv | `dim_customers` | Filter `is_active == 1` |
| `dfdimaccounts` | bank_accounts.csv | `dim_accounts` | Derive `account_age_years`, `missing_nominee_flag` |
| `dfdimemployees` | bank_employees.csv | `dim_employees` | Derive `is_manager`, `certification_count` |
| `dfdimbranches` | 3 CSV sources | `dim_branches` | Union → Aggregate distinct `branch_id` |
| `dfdimloantypes` | bank_loans.json | `dim_loan_types` | Aggregate distinct `loan_type` |

---

#### 2️⃣ `pl_load_fact_transactions`
Flattens nested **metadata JSON** object from transactions.

```
// Key Derived Columns:
metadeviceid       = metadata.device_id
metaipaddress      = metadata.ip_address
metalocationcity   = metadata.location_city
metaprocessingbank = metadata.processing_bank
metabatchid        = metadata.batch_id
isinternaltransfer = iif(isNull(beneficiary_account), 1, 0)
```

---

#### 3️⃣ `pl_load_fact_loans`
Boolean conversion + NPA tier classification.

```
// Insurance flag:
insurancelinkedflag = iif(insurance_linked == true(), 'Yes', 'No')

// Overdue category:
overduecategory = iif(emis_overdue == 0, 'Current',
  iif(emis_overdue <= 2, 'Special Mention',
    iif(emis_overdue <= 4, 'Sub-Standard', 'Doubtful')))
```

---

#### 4️⃣ `pl_load_fraud_and_balances`
Two branches running **in parallel**.

```
dffactfraudalerts      ─┐  PARALLEL
                         ├──→ Both load simultaneously
dffactaccountbalances  ─┘
```

---

#### 5️⃣ `pl_build_aggregations`
Reads from **warehouse tables only** (not source files).

| Data Flow | Group By | Target |
|-----------|----------|--------|
| `dffaggbranchperformance` | `branch_id` | `agg_branch_performance` |
| `dffaggcustomer360` | `customer_id` | `agg_customer_360` |
| `dffaggfraudsummary` | `severity + month + year` | `agg_fraud_summary` |

---

#### 6️⃣ `pl_master_bank_etl`
Master orchestration pipeline with failure dependencies.

| Stage | Pipeline | Duration |
|-------|----------|----------|
| Stage 1 | `pl_load_dimensions` | 5m 39s ✅ |
| Stage 2 | `pl_load_fact_transactions` | 26s ✅ |
| Stage 3A | `pl_load_fact_loans` | 4m 24s ✅ |
| Stage 3B | `pl_load_fraud_and_balances` | 42s ✅ |
| Stage 4 | `pl_build_aggregations` | 1m 37s ✅ |

---

## ✅ Row Count Validation

| # | Table | Type | Rows |
|---|-------|------|------|
| 1 | `dim_customers` | Dimension | 1,328 |
| 2 | `dim_accounts` | Dimension | 2,200 |
| 3 | `dim_employees` | Dimension | 400 |
| 4 | `dim_branches` | Dimension | 40 |
| 5 | `dim_loan_types` | Dimension | 9 |
| 6 | `fact_transactions` | Fact | 3,000 |
| 7 | `fact_loans` | Fact | 1,200 |
| 8 | `fact_fraud_alerts` | Fact | 600 |
| 9 | `fact_account_balances` | Fact | 2,200 |
| 10 | `agg_branch_performance` | Aggregation | 40 |
| 11 | `agg_customer_360` | Aggregation | 1,328 |
| 12 | `agg_fraud_summary` | Aggregation | 112 |
| | **TOTAL** | | **12,457** |

---

## 🔒 Security

- ✅ Azure Key Vault for all credentials
- ✅ Azure RBAC permission model
- ✅ ADF Managed Identity enabled
- ✅ No hardcoded passwords anywhere
- ✅ Key Vault Secrets Officer role assigned

---

## ⏰ Triggers

| Trigger | Type | Schedule |
|---------|------|----------|
| `tr_daily_1am_ist` | Schedule | Daily at 1:00 AM IST |
| `tr_fraud_upload_event` | Storage Event | New CSV in `/fraud-uploads/` |

---

## 📊 Monitoring

- **Azure Monitor** alert rules configured (pipeline failure + timeout)
- **Log Analytics** workspace for pipeline diagnostics
- **Diagnostic categories**: Pipeline runs, Activity runs, Trigger runs

---

## 🚀 How to Deploy

```bash
# Step 1: Clone this repository
git clone https://github.com/your-username/NBI-ADF-Banking-Project.git

# Step 2: Create Azure Resources
# - Resource Group: rg-nbi-banking
# - ADF, Blob Storage, SQL Database, Key Vault

# Step 3: Upload source files to Blob Storage
# Container: source-data

# Step 4: Run SQL script to create target tables
# Execute: sql-scripts/NBI_ADF_Target_Tables.sql

# Step 5: Import ADF ARM Template
# ADF Studio → Manage → ARM Template → Import

# Step 6: Update Linked Service credentials

# Step 7: Run pl_master_bank_etl pipeline
```

---

## 📚 Technologies Used

- **Azure Data Factory** — ETL orchestration, Data Flows, Pipelines
- **Azure Blob Storage** — Source data lake (CSV + JSON)
- **Azure SQL Database** — Target data warehouse
- **Azure Key Vault** — Secret management
- **Azure Monitor** — Alerting and monitoring
- **Log Analytics** — Diagnostic logging
- **SQL** — Schema creation, validation queries
- **ADF Expression Language** — Data transformations

---

## 👤 Author

**Rydham Jadhav**
- 🎓 Svit Vasad

---

> ⭐ **If this project helped you, please star the repository!**
