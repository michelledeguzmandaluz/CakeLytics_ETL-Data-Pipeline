# Stage 1: Data Extraction Documentation

## Objective
The goal of this stage is to extract data from an identified source system and prepare a documented, traceable, and validated dataset for the next stage of the data pipeline.

---

## 1. Data Source and Extraction Specification

### Source System
* **Name & Purpose:** CakeLytics Point-of-Sale (POS) and Inventory Management System. Used for managing day-to-day order processing, sales, material usage, and waste logging.

### Source Database or File
* **DBMS:** Supabase PostgreSQL (Cloud Database).
* **Format:** Relational PostgreSQL Tables.

### Extraction Method
* **Tool / Script:** Python script using SQLAlchemy ORM with the `pg8000` native PostgreSQL driver.
* **Orchestration:** Scheduled and executed via Prefect flows.
* **Extraction Mode:** Full Extraction (Baseline).
* **Extraction Schedule:** Daily at 12:00 AM midnight.
* **Cron Expression:** `0 0 * * *`
* **GitHub Issue:** `[PENDING ISSUER ASSIGNMENT]` (To be linked upon creation of the extraction issue by the team Issuer).

### Extraction Scope
* `order_items` inner joined with `orders` for itemized sales records.
* `waste_logs` for operational waste expense logs.
* `raw_ingredients` and `celebration_materials` for inventory stock levels and unit costs.

### Source Limitations and Assumptions
* **Database Availability:** Extraction requires stable internet connectivity and valid Supabase credentials stored securely as environment variables.
* **Relational Dependencies:** Relies on valid relational integrity between `orders.id` and `order_items.order_id`.
* **Scope Limitation:** Profit is not extracted directly from the source; it will be calculated downstream during the Transformation stage. Data extraction contains zero business transformations or calculated metrics.

---

## 2. Source Tables and Column Specification

### Table 1: `orders`
* **Purpose:** Stores overall order header details, order status, and transaction timestamps.
* **Primary Key:** `id` (UUID)
* **Reason for Inclusion:** Required to determine daily sales timestamps (`created_at`) and overall transaction validity.

| Column Name | Data Type | Key / Constraint | Reason for Inclusion |
| :--- | :--- | :--- | :--- |
| `id` | UUID | Primary Key | Unique transaction identifier. |
| `order_number` | TEXT | Unique / Not Null | Human-readable tracking number (e.g., ORD-0001). |
| `status` | order_status ENUM | Not Null | Filters completed/confirmed orders from cancelled ones. |
| `grand_total` | NUMERIC(10,2) | Default 0 | Order-level total revenue amount. |
| `created_at` | TIMESTAMPTZ | Not Null | Primary reporting timestamp for daily aggregation. |

### Table 2: `order_items`
* **Purpose:** Contains itemized product lines associated with each sales order.
* **Primary Key:** `id` (UUID)
* **Foreign Key:** `order_id` references `orders(id)`
* **Reason for Inclusion:** Provides itemized sales records for product-level revenue analysis.

| Column Name | Data Type | Key / Constraint | Reason for Inclusion |
| :--- | :--- | :--- | :--- |
| `id` | UUID | Primary Key | Unique line-item identifier. |
| `order_id` | UUID | Foreign Key (`orders.id`) | Links line item directly to parent order. |
| `product_id` | UUID | Foreign Key (`products.id`) | Identifies the catalog product sold. |
| `product_name` | TEXT | Not Null | Historical snapshot of product name at time of sale. |
| `quantity` | INT | Check (>0) | Number of units purchased. |
| `unit_price` | NUMERIC(10,2) | Check (>=0) | Price charged per single unit. |
| `total_price` | NUMERIC(10,2) | Check (>=0) | Calculated line total (quantity * unit_price). |

### Table 3: `waste_logs`
* **Purpose:** Tracks raw ingredients, packaging, or spoiled goods logged as operational loss.
* **Primary Key:** `id` (UUID)
* **Reason for Inclusion:** Essential for calculating daily operational waste expenses.

| Column Name | Data Type | Key / Constraint | Reason for Inclusion |
| :--- | :--- | :--- | :--- |
| `id` | UUID | Primary Key | Unique waste entry identifier. |
| `waste_type` | waste_type ENUM | Not Null | Categorizes loss (ingredient, material, product). |
| `item_name` | TEXT | Not Null | Name of wasted inventory item. |
| `quantity` | NUMERIC(12,4) | Check (>0) | Amount or volume discarded. |
| `unit` | TEXT | Not Null | Unit of measure (e.g., kg, pcs, liters). |
| `reason` | TEXT | Not Null | Explanation of loss recorded. |
| `notes` | TEXT | Default '' | Additional operational remarks. |
| `cost` | NUMERIC(10,2) | Default 0 | Financial expense assigned to the wasted items. |
| `logged_at` | TIMESTAMPTZ | Not Null | Timestamp when loss was recorded. |

### Table 4: `raw_ingredients`
* **Purpose:** Stores inventory stock levels, threshold limits, and unit purchasing costs for raw production ingredients.
* **Primary Key:** `id` (UUID)
* **Reason for Inclusion:** Provides ingredient unit cost metrics for total expense calculations in downstream transformations.

| Column Name | Data Type | Key / Constraint | Reason for Inclusion |
| :--- | :--- | :--- | :--- |
| `id` | UUID | Primary Key | Unique raw ingredient identifier. |
| `name` | TEXT | Unique / Not Null | Name of the raw ingredient. |
| `unit` | TEXT | Not Null | Unit of measurement (e.g., kg, liters). |
| `stock_quantity` | NUMERIC(12,4) | Check (>=0) | Current inventory on-hand quantity. |
| `minimum_stock` | NUMERIC(12,4) | Check (>=0) | Minimum stock threshold before alert. |
| `cost_per_unit` | NUMERIC(10,4) | Check (>=0) | Unit purchasing cost used to determine raw material cost. |
| `updated_at` | TIMESTAMPTZ | Not Null | Timestamp when stock or cost was last updated. |

### Table 5: `celebration_materials`
* **Purpose:** Tracks non-food materials, packaging, and add-ons used for orders.
* **Primary Key:** `id` (UUID)
* **Reason for Inclusion:** Required to calculate non-ingredient operational material costs in expense aggregation.

| Column Name | Data Type | Key / Constraint | Reason for Inclusion |
| :--- | :--- | :--- | :--- |
| `id` | UUID | Primary Key | Unique celebration material identifier. |
| `name` | TEXT | Unique / Not Null | Name of celebration material/packaging. |
| `unit` | TEXT | Not Null | Unit of measure (e.g., pc, set, pcs). |
| `stock_quantity` | NUMERIC(12,4) | Check (>=0) | Current material stock quantity on-hand. |
| `minimum_stock` | NUMERIC(12,4) | Check (>=0) | Reorder threshold stock quantity. |
| `cost_per_unit` | NUMERIC(10,4) | Check (>=0) | Unit purchasing cost for material expense tracking. |
| `updated_at` | TIMESTAMPTZ | Not Null | Timestamp when material stock or cost was updated. |

---

## 3. Extraction Validation and Data Quality Checks

The following validation checks ensure that extracted data is complete and accurate before passing to the Transformation stage:

| Check Name | Target | Purpose | Validation Criteria | Verification Status |
| :--- | :--- | :--- | :--- | :--- |
| **Database Connectivity** | Supabase Engine | Verify database accessibility before query execution. | SQLAlchemy connection ping succeeds without timeout. | `[PENDING EXECUTION]` |
| **Schema Integrity** | Source Tables & Columns | Prevent pipeline breaks caused by missing or renamed columns. | All expected target tables and specified columns exist. | `[PENDING EXECUTION]` |
| **Primary Key Uniqueness** | `id` columns | Guarantee no duplicate records enter the pipeline. | `COUNT(id) == COUNT(DISTINCT id)` evaluates to true. | `[PENDING EXECUTION]` |
| **Record Count Verification** | Source vs Extracted | Ensure total record completeness during fetch. | Extracted DataFrame row count matches database result count. | `[PENDING EXECUTION]` |
| **Sales Join Consistency** | `order_items` × `orders` | Detect broken foreign key relationships. | Zero orphan `order_items` where `order_id` fails to map. | `[PENDING EXECUTION]` |

---

## 4. Extraction Metadata and Log Specification

Each pipeline execution logs metadata for run monitoring, troubleshooting, auditing, and recovery:

| Field Name | Purpose | Specification / Format | Verification Status |
| :--- | :--- | :--- | :--- |
| `run_id` | Unique execution instance identifier | Format: `EXT-YYYYMMDD-XXX` | `[PROPOSED]` |
| `table_name` | Target table or joined query scope | String (e.g., `order_items_joined`) | `[PROPOSED]` |
| `start_time` | Extraction start timestamp | ISO 8601 Timestamp | `[PROPOSED]` |
| `end_time` | Extraction completion timestamp | ISO 8601 Timestamp | `[PROPOSED]` |
| `status` | Execution outcome state | Enum: `SUCCESS` / `FAILED` | `[PROPOSED]` |
| `records_read` | Total rows queried from database | Integer | `[PROPOSED]` |
| `records_extracted` | Total valid rows fetched into memory | Integer | `[PROPOSED]` |
| `records_rejected` | Rows excluded due to validation errors | Integer | `[PROPOSED]` |
| `validation_result` | Overall check status | Enum: `PASS` / `WARNING` / `FAIL` | `[PROPOSED]` |
| `error_message` | Detailed trace if failure occurs | String / Text | `[PROPOSED]` |
| `duration_seconds` | Total execution run time duration | Float (Seconds) | `[PROPOSED]` |

### Log Usage
Metadata and execution logs are processed through Prefect Cloud to allow real-time monitoring, trigger retries on failure, and audit daily extraction trends.

---

## 5. Log Retention and Access

* **Log Retention Period & Justification:** Operational logs will be retained for **90 days** `[PROPOSED]` to support operational troubleshooting and quarterly auditing without overloading storage resources.
* **Storage Location:** Prefect Cloud dashboard and local repository log path `[TO BE CONFIRMED UPON IMPLEMENTATION]`.
* **Access Permissions:** Restricted to Database Administrators (DBAs) and Data Engineers.
* **Archiving & Cleanup Rules:** Logs older than 90 days are automatically purged via scheduled cleanup scripts, provided no active audit flag is attached.

---

## 6. Extraction Data Contract and Naming Convention

### Naming Convention
All extracted source tables and fields follow standard lower `snake_case` naming format. Primary keys use `id` and foreign keys follow `[table_singular]_id`.

### Source-to-Standardized Column Mapping (Extract Staging in Pandas)
The target types represent the in-memory Pandas DataFrame staging types used after extraction before transformation:

| Source Column | Standardized Column | Source Type | Target Type (Pandas Staging) | Mapping Type |
| :--- | :--- | :--- | :--- | :--- |
| `id` | `item_id` | UUID | STRING | Rename |
| `order_id` | `order_id` | UUID | STRING | Type Mapping |
| `total_price` | `item_total_price` | NUMERIC | FLOAT | Rename & Type Mapping |
| `created_at` | `transaction_timestamp` | TIMESTAMPTZ | TIMESTAMP | Rename & Type Mapping |
| `logged_at` | `waste_timestamp` | TIMESTAMPTZ | TIMESTAMP | Rename & Type Mapping |

### Schema Consistency
If unexpected schema drift occurs (e.g., column renamed in Supabase), the pre-extraction validation step halts execution and sends an alert without corrupting downstream stages.

---

## 7. Extraction Acceptance and Handover Rules

### Decision Rules

| Overall Condition | Acceptance Decision | Pipeline Action | Decision Type | Responsible Role |
| :--- | :--- | :--- | :--- | :--- |
| All required validation checks PASS | **APPROVED** | Proceed to Transformation Stage | Automatic | System / Prefect |
| Non-critical warning (e.g., zero waste logged) | **MANUAL REVIEW** | Hold pipeline execution | Manual Review | Data Engineer |
| One or more critical checks FAIL | **REJECTED** | Halt pipeline; retry 2x with 60s delay | Automatic / Retry | System / Prefect |

### Validation Summary Handover Baseline

| Validation Target | Current Verification Status |
| :--- | :--- |
| `orders` table validation | `[PENDING TESTING]` |
| `order_items` table validation | `[PENDING TESTING]` |
| `waste_logs` table validation | `[PENDING TESTING]` |
| `raw_ingredients` table validation | `[PENDING TESTING]` |
| `celebration_materials` table validation | `[PENDING TESTING]` |
| **Overall Extraction Handover** | `[PENDING EXECUTION]` |

### Handover
Upon approval, the raw data structures, execution metadata, and validation summary will be passed downstream to the **Transformation Stage**.

---

## 8. Common Extraction Problems and Mitigation

| Problem Description | Possible Cause | Impact | Proposed Mitigation |
| :--- | :--- | :--- | :--- |
| **Database Unreachable** | Network disruption or Supabase database sleep/maintenance state. | Extraction process halts completely. | Prefect automatically retries execution 2 times with a 60-second delay before sending alert logs. |
| **Schema Drift / Column Missing** | Unannounced modification to source tables in Supabase. | Query fails or produces null/corrupted fields. | Pre-extraction schema assertion stops execution immediately to prevent passing invalid fields downstream. |
| **Incomplete Join Results** | Orphaned `order_items` records missing parent `orders` entries. | Inaccurate financial calculations. | Validation assertions quarantine unlinked rows and mark the execution status for manual review. |