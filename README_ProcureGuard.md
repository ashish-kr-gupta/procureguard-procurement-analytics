# ProcureGuard — Procurement Contract Compliance & Supplier Analytics

> **Portfolio project | Excel · Power Query · Power BI · DAX**
>
> This project uses entirely fictional/synthetic data for learning and demonstration. It is not an implementation for a real company and does not establish actual savings or confirmed supplier violations.

## 1. Business problem

Procurement teams need a consistent way to monitor contract terms, supplier invoices, renewal dates, and contractual deliverables. Spreadsheets stored separately can make exceptions difficult to spot and follow up.

ProcureGuard is a Power BI reporting concept that brings four related datasets together and highlights:
- Invoice lines where the invoiced unit rate exceeds the reference contract rate.
- Contracts approaching expiry.
- Outstanding and overdue contractual obligations.
- Spend by supplier and procurement category.

## 2. Dataset

`ProcureGuard_Procurement_Dataset.xlsx` contains five sheets:

| Sheet | Grain | Purpose |
|---|---|---|
| `Suppliers` | One row per supplier | Supplier, category, region, status |
| `Contracts` | One row per contract | Contract value, dates, rate, payment terms and status |
| `Transactions` | One row per invoice line | PO/invoice references, quantity, contract rate, invoiced rate, amount and compliance flag |
| `Obligations` | One row per contractual obligation | Deliverable, due date, completion date, status and priority |
| `Metric_Checks` | One row per control metric | Reconciliation targets and definitions |

**Snapshot date:** 31 January 2025. Expiry and overdue classifications use this fixed date so the sample report is reproducible.

### Reconciled headline figures

- Total contract value: **₹12,500,000 (₹12.5M)**
- Total invoiced spend: **₹9,800,000 (₹9.8M)**
- Contracts: **43 total; 34 active, 3 expired, 6 closed**
- Active contracts expiring within 90 days: **6** (included in the 34 active contracts)
- Invoice lines: **142 total; 118 compliant; 24 flagged**
- Compliance rate: **118 / 142 = 83.1%**
- Potential positive rate variance: **₹42,000**
- Obligations: **78 total; 56 completed, 8 in progress, 14 overdue**
- Completion rate: **56 / 78 = 71.8% (72% rounded)**

The ₹42,000 is a **potential rate discrepancy for investigation**, not confirmed overbilling or realized savings. Check contract amendments, taxes, currency, units of measure, and approved charges before drawing conclusions.

## 3. End-to-end workflow

### Step 1 — Define business questions
1. Are invoiced unit rates consistent with reference contract rates?
2. Which active contracts end within 30/60/90 days?
3. Which contractual obligations are overdue?
4. Which suppliers/categories have more flagged invoice lines?
5. What spend and contract value should be prioritized for review?

### Step 2 — Import the Excel workbook into Power BI Desktop
1. Open **Power BI Desktop**.
2. Select **Home → Get data → Excel workbook**.
3. Select `ProcureGuard_Procurement_Dataset.xlsx`.
4. In Navigator, select `Suppliers`, `Contracts`, `Transactions`, and `Obligations` (optionally `Metric_Checks` for QA).
5. Choose **Transform Data** rather than immediately loading, so the queries can be inspected and cleaned.

### Step 3 — Clean data in Power Query
For each query:
- Set IDs and status fields to Text.
- Set contract, invoice and due/completion dates to Date.
- Set quantity to Whole Number; rates, invoice amounts and contract value to Decimal Number or Fixed Decimal Number.
- Check required IDs for nulls and check duplicate keys in the relevant dimension tables.
- Standardize status spelling and whitespace.
- Validate that every transaction and obligation references a known Contract ID.
- Keep the data at its intended grain; do not merge transactions and obligations into a single table because this can multiply amounts.

Select **Close & Apply** when checks are complete.

### Step 4 — Create the data model
Create these relationships (one-to-many, single direction from parent to child):
- `Suppliers[Supplier ID]` → `Contracts[Supplier ID]`
- `Contracts[Contract ID]` → `Transactions[Contract ID]`
- `Contracts[Contract ID]` → `Obligations[Contract ID]`

The sample contains one supplier per contract and one row per contract in `Contracts`. Check relationship cardinality and unmatched IDs in Model view. Avoid many-to-many relationships unless there is a documented reason.

### Step 5 — Add DAX measures

```DAX
Total Contract Value =
SUM(Contracts[Contract Value (INR)])

Total Invoiced Spend =
SUM(Transactions[Invoice Amount (INR)])

Total Invoice Lines =
COUNTROWS(Transactions)

Compliant Invoice Lines =
CALCULATE(
    [Total Invoice Lines],
    Transactions[Compliance Status] = "Compliant"
)

Non-Compliant Invoice Lines =
CALCULATE(
    [Total Invoice Lines],
    Transactions[Compliance Status] = "Non-Compliant"
)

Compliance Rate =
DIVIDE([Compliant Invoice Lines], [Total Invoice Lines], 0)

Potential Rate Variance =
SUM(Transactions[Rate Variance (INR)])

Total Contracts =
DISTINCTCOUNT(Contracts[Contract ID])

Active Contracts =
CALCULATE(
    [Total Contracts],
    Contracts[Contract Status] = "Active"
)

Contracts Expiring in 90 Days =
CALCULATE(
    [Total Contracts],
    Contracts[Expiring in 90 Days] = "Yes"
)

Total Obligations =
COUNTROWS(Obligations)

Completed Obligations =
CALCULATE(
    [Total Obligations],
    Obligations[Obligation Status] = "Completed"
)

In Progress Obligations =
CALCULATE(
    [Total Obligations],
    Obligations[Obligation Status] = "In Progress"
)

Overdue Obligations =
CALCULATE(
    [Total Obligations],
    Obligations[Obligation Status] = "Overdue"
)

Obligation Completion Rate =
DIVIDE([Completed Obligations], [Total Obligations], 0)
```

Format currency measures as INR and rate measures as percentages. The `Potential Rate Variance` measure sums positive and negative differences if any exist; in this sample all flagged variance values are positive.

### Step 6 — Build the report pages

**Page 1: Procurement Overview**
- KPI cards: contract value, invoiced spend, active contracts, suppliers, contracts expiring within 90 days.
- Donut: spend by category.
- Bar chart: top suppliers by spend.
- Line chart: monthly invoiced spend.
- Column chart: contract status.
- Date slicers: contract start/end date, if implemented using a suitable date model.

**Page 2: Contract Compliance**
- KPI cards: invoice lines, compliant lines, non-compliant lines, compliance rate, potential rate variance.
- Stacked bar: compliant vs. flagged invoice lines by supplier.
- Clustered column chart: agreed vs. invoiced rates for selected sample items.
- Table: invoice, contract, supplier, item, agreed rate, invoiced rate, variance and status.
- Slicers: supplier, category and contract ID.

**Page 3: Contract & Obligations Tracker**
- KPI cards: total contracts, active contracts, expiring contracts, total obligations, completed, in progress, overdue and completion rate.
- Table of contracts ending within 90 days, with days remaining.
- Stacked bar: obligation statuses by supplier.
- Donut: obligations by status.
- Slicers: supplier, contract status and obligation status.

### Step 7 — Validate and reconcile
Use the `Metric_Checks` sheet to confirm:
- Contract values sum to ₹12.5M.
- Invoice amounts sum to ₹9.8M.
- 118 + 24 = 142 invoice lines and compliance is 83.1%.
- 34 active + 3 expired + 6 closed = 43 contracts.
- Six contracts are expiring within 90 days and are included in active contracts.
- 56 + 8 + 14 = 78 obligations; completion rate is 71.8%.

Reconcile invoice totals against Excel, manually recalculate a few variances, test slicers, and inspect relationship behavior. The fixed snapshot date is used for reproducibility; a production dashboard should use a refreshable reporting date or calendar table.

### Step 8 — Write findings and recommendations
- Route unexplained invoice rate differences for contract/amendment verification.
- Start renewal review for contracts expiring soon.
- Follow up on overdue supplier deliverables and record action owners.
- Review recurring supplier exceptions with contract owners.
- State that the dashboard flags exceptions; it does not itself prove a breach, approve payment, or calculate realized savings.

## 4. Suggested repository structure

```text
ProcureGuard/
├── data/
│   └── ProcureGuard_Procurement_Dataset.xlsx
├── report/
│   └── ProcureGuard_Project_Report.docx
├── dashboard/
│   └── Procurement_Dashboard.png
└── README.md
```

Save your actual Power BI `.pbix` file in `dashboard/` after building the report. The screenshot is a design reference; it is not an interactive PBIX file.

## 5. Resume bullets (use after completing the work)

**ProcureGuard — Procurement Contract Compliance & Supplier Analytics | Excel, Power Query, Power BI, DAX**
- Built an interactive Power BI reporting solution using synthetic supplier, contract, invoice and obligation datasets to monitor spend, invoice rate variances, contract expirations and overdue deliverables.
- Cleaned and transformed data in Power Query, modelled related procurement tables, and developed DAX KPIs for invoice compliance, potential rate variance and contractual obligation completion.
- Designed overview, compliance and contract-tracking report pages with supplier-level analysis and exception tables; reconciled headline metrics against source data and documented business recommendations.

Be transparent that the data is synthetic and do not claim real savings or a real client implementation.

## 6. Interview talking points
- Why you separated the tables and used IDs for relationships.
- Difference between contract rate compliance and three-way matching.
- Why a rate variance is a potential exception, not automatically confirmed overbilling.
- How you reconciled totals and prevented duplicated amounts.
- How the dashboard helps prioritize contract renewals and supplier follow-ups.

## 7. Limitations and next improvements
- Data is synthetic and intentionally designed to illustrate the workflow.
- Current compliance flags are pre-labelled for teaching; in a production solution, derive them from approved contract terms, amendments and business rules.
- The sample does not include a full purchase order, receipt and invoice matching process.
- Future enhancements: a proper Date table, configurable tolerance thresholds, exception ownership, refresh monitoring and row-level security.

---
**License suggestion:** MIT for the project documentation and model code, if you choose to make the repository public. State that the dataset is synthetic.
