# DKT ERP — Manufacturing & Distribution Management System

> **Note:** This repository contains no application code. It is an architecture reference document describing the system design, technical decisions, and infrastructure of a production multi-tenant financial SaaS platform I built as Lead Full-Stack Engineer.

---

![Python](https://img.shields.io/badge/Python-3.8-3776AB?style=for-the-badge&logo=python&logoColor=white)
![Tornado](https://img.shields.io/badge/Tornado-Async%20Web%20Framework-F7931E?style=for-the-badge)
![Angular](https://img.shields.io/badge/Angular-12+-DD0031?style=for-the-badge&logo=angular&logoColor=white)
![Angular Material](https://img.shields.io/badge/Angular%20Material-UI-757575?style=for-the-badge&logo=material-design&logoColor=white)
![MySQL](https://img.shields.io/badge/MySQL-Database-4479A1?style=for-the-badge&logo=mysql&logoColor=white)
![SQLAlchemy](https://img.shields.io/badge/SQLAlchemy-ORM-CC2927?style=for-the-badge)
![Docker](https://img.shields.io/badge/Docker-Containerized-2496ED?style=for-the-badge&logo=docker&logoColor=white)
![JWT](https://img.shields.io/badge/JWT-Auth-000000?style=for-the-badge&logo=json-web-tokens&logoColor=white)
![ReportLab](https://img.shields.io/badge/ReportLab-PDF%20Generation-EE4C2C?style=for-the-badge)

---

## Overview

DKT ERP is a full-stack, multi-tenant Enterprise Resource Planning system purpose-built for **manufacturing and field-distribution businesses**. The client's core problem was managing the entire product lifecycle — from raw material procurement through blending and production, out to field sales teams operating daily routes, and back to finance with payroll and PDF reporting — all in disconnected spreadsheets and paper records.

The system replaces that with a single, role-protected web application covering seven integrated modules: **Setup, Inventory, Purchases, Manufacturing, Sales, HRM, and Reports**. Each company gets its own isolated database instance, so the platform supports multiple tenants with zero data bleed between them.

---

## Tech Stack

| Layer             | Technology                                  |
| ----------------- | ------------------------------------------- |
| Backend framework | Python 3.8 + Tornado (async)                |
| ORM               | SQLAlchemy with async query execution       |
| Database          | MySQL (per-tenant isolated databases)       |
| Frontend          | Angular 12+ (two SPA apps: `auth`, `setup`) |
| UI library        | Angular Material (Material Design)          |
| Authentication    | JWT (PyJWT) + server-side session table     |
| PDF generation    | ReportLab + SVGLib                          |
| RTL / Arabic text | arabic-reshaper + python-bidi               |
| Email             | SendGrid                                    |
| Encryption        | PyCryptodome                                |
| Containerization  | Docker (Python 3.8 Alpine)                  |

---

## Architecture

The application follows a **decoupled SPA + REST API** architecture:

- **`auth`** — standalone Angular SPA for login, registration, and password reset
- **`setup`** — main Angular SPA containing all ERP modules, route-guarded by role permissions
- **Tornado backend** — async REST API server serving both SPAs and all API endpoints
- **Multi-tenant isolation** — each registered company receives a dedicated MySQL database, provisioned at company creation time; the central `companies` table stores credentials and routing info

```
Browser
  ├── auth SPA  →  /auth/*  endpoints
  └── setup SPA →  /api/*  endpoints (JWT-protected)
          │
          ▼
    Tornado Async Server
          │
    ┌─────┴──────┐
    │  Central DB │   (users, companies, roles)
    └─────────────┘
    ┌─────┴──────┐
    │ Company DB  │   (per-tenant: inventory, sales, HR, …)
    └─────────────┘
```

---

## Module Walkthrough

### 1. Setup & Administration

<!-- ![Setup](./screenshots/setup.png) -->

The entry point for system configuration. Administrators manage:

- **Roles** — granular boolean permissions per module (expenses, purchases, sales, blending, production, stock transfer, routes, user management)
- **Users** — create users and bind them to roles and a specific company
- **Banks** — bank account records linked to supplier payment workflows
- **Areas** — geographic sales territories, each assigned to a responsible salesperson and carrying a `market_day` schedule
- **Stores** — physical warehouse/store locations used across inventory, production, and stock transfer

Role permissions are enforced at the API handler level via `SecureApiHandler`, meaning unauthorised users cannot call restricted endpoints regardless of frontend state.

---

### 2. Inventory Management

<!-- ![Inventory](./screenshots/inventory.png) -->

Tracks the full product catalogue with separation between **finished goods** and **raw materials**:

- **Categories & Brands** — hierarchical product classification
- **Products** — each product record carries purchase price, sale price, units-per-carton, kg weight, reorder level, low-stock threshold, a default supplier link, and an `is_raw_material` flag that controls which stock pool it draws from
- **Stock history** — every stock movement (purchase, production, blending, transfer, sale) is recorded in `StoreItemHistory` and `RawItemHistory` tables with a `transaction_type` tag, enabling point-in-time stock reconstruction

---

### 3. Purchases Module

<!-- ![Purchases](./screenshots/purchases.png) -->

Handles the complete procure-to-pay cycle:

| Sub-module         | Description                                                                                                                    |
| ------------------ | ------------------------------------------------------------------------------------------------------------------------------ |
| Suppliers          | Full contact records with auto-incremented supplier code and running balance tracking                                          |
| Purchase / GRN     | Multi-line purchase orders with per-item discount (% or flat amount), unit cost calculation, automatic stock increment on save |
| Purchase Returns   | Partial or full returns against existing purchases with reason codes                                                           |
| Payments           | Supplier payment records supporting cash and bank transfers; balance maintained via running `fetch_supplier_balance` helper    |
| Expenses           | Operational expense entries against configurable categories                                                                    |
| Expense Categories | Custom category taxonomy for expense classification                                                                            |

---

### 4. Manufacturing Module

<!-- ![Manufacturing](./screenshots/manufacturing.png) -->

This module was one of the more technically complex requirements, handling a two-stage manufacturing process:

**Blending**
Raw materials are combined into intermediate blended products. Each blending record captures input materials (with quantities and costs), assigns a unique `batch_no`, and decrements raw material stock while incrementing blended product stock.

**Production**
Blended or raw materials are converted into finished goods. Production records track:

- Bill of materials (input items with quantities and unit costs)
- Output product, quantity, and carton quantity
- Material cost + other expenses to compute unit cost and unit price
- Batch number and expiry date for full traceability
- Target store for the finished stock

**Stock Transfers**
Finished goods can be moved between stores. Each transfer records source store, destination store, items with quantities and unit cost/price, and updates both store inventory ledgers.

---

### 5. Sales Module

<!-- ![Sales](./screenshots/sales.png) -->

Manages the full field-sales workflow from customer to cash:

| Sub-module              | Description                                                                                                      |
| ----------------------- | ---------------------------------------------------------------------------------------------------------------- |
| Customers               | Area-linked customer records with opening balance and last-visit tracking                                        |
| Sales                   | Multi-line invoices with per-order discount, received amount, and automatic customer balance update              |
| Sale Returns            | Partial or full return against existing invoices                                                                 |
| Receipts                | Cash recovery entries by salesperson against customer accounts                                                   |
| Daily Routes            | Planned field routes per salesperson; each visit logs sale total, recovery, return, and running customer balance |
| Pending Cash            | Dashboard of outstanding customer balances per salesperson                                                       |
| Pending Stocks          | Tracks undelivered stock in the sales pipeline                                                                   |
| Store Cash Transactions | Per-store cash ledger for end-of-day reconciliation                                                              |

---

### 6. HRM & Payroll

<!-- ![HRM](./screenshots/hrm.png) -->

A purpose-built HR module tightly integrated with the sales performance data:

| Feature           | Details                                                                                                                  |
| ----------------- | ------------------------------------------------------------------------------------------------------------------------ |
| Employees         | Full profile with designation, store assignment, join date; optionally linked to an app user account                     |
| Monthly Targets   | Per-employee recovery targets used for incentive calculation                                                             |
| Sale Commissions  | Per-category commission entries per employee per month                                                                   |
| Bonuses           | One-off or recurring bonus records                                                                                       |
| Deductions        | Deduction entries (advances, penalties, etc.)                                                                            |
| Loans             | Loan disbursement records with repayment tracking                                                                        |
| Salary Sheets     | Monthly payroll computation aggregating basic pay, meal allowance, bonuses, commissions, deductions, and loan repayments |
| Employee Salaries | Individual salary disbursement records per payroll run                                                                   |

The **incentive engine** computes tiered bonuses from salesperson recovery vs. target: a 3,000 base bonus at 100–125% attainment, scaling up by 5,000 + 100 per percentage point for every additional 25% band beyond that.

---

### 7. Reports Module

<!-- ![Reports](./screenshots/reports.png) -->

PDF report generation using **ReportLab** with full **Arabic/Urdu RTL text** support (arabic-reshaper + python-bidi), enabling bilingual output for the client's local market.

| Report                | Content                                                                  |
| --------------------- | ------------------------------------------------------------------------ |
| Stock Report          | Opening stock, purchases, sales, and closing stock per product per store |
| Raw Stock Report      | Raw material stock levels                                                |
| Sale Report           | Itemised sales by date range and salesperson                             |
| Area Report           | Sales performance broken down by geographic area                         |
| Customer Report       | Customer-level balance and transaction history                           |
| Market Balance Report | Outstanding balances across all customers by market                      |
| Monthly Report        | Month-over-month sales and recovery summary                              |
| Expense Report        | Expenses by category and date range                                      |
| Online Payment Report | Bank/online payment records for the period                               |
| Salary Sheet Report   | Formatted payroll sheet for a selected month                             |
| Payslip               | Individual employee payslip PDF                                          |
| Store Cash Report     | Store-level cash summary                                                 |
| Store Stock Report    | Per-store stock valuation                                                |

All reports share a common `CHIReportTemplate` base class and a custom `CHIFrame`/`CHIPaint` rendering layer built on top of ReportLab's canvas API.

---

## Key Technical Highlights

### Async Backend

The entire Tornado backend is async end-to-end — handlers, database queries, and helpers all use `async/await`. This allows the single-threaded server to handle concurrent requests without blocking on I/O, which is significant for report generation jobs that can take several seconds.

### Multi-Tenant Isolation

Company data never shares a database schema. At company registration, the server dynamically creates a new MySQL database, provisions a dedicated user, and runs `DBBase.metadata.create_all()` against it. The central `companies` table stores only the connection credentials. Every authenticated request resolves the correct database engine from the user's session.

### Role-Based Access Control

All protected API routes inherit from `SecureApiHandler`. On each request it validates the JWT, resolves the user's role, and checks the relevant boolean permission column before executing the handler method. The Angular frontend mirrors this with route guards (`AppRoutesGuard`) that hide menu items the user lacks permissions for.

### PDF Generation with Arabic RTL Support

ReportLab does not natively support Arabic/Urdu script. The `chi_report_template.py` layer applies `arabic-reshaper` to re-join disconnected Arabic glyphs and `python-bidi` to apply the Unicode Bidirectional Algorithm before passing text to ReportLab's paragraph engine — producing correctly rendered right-to-left text in the generated PDFs.

### Immutable Stock Ledger

Stock levels are never stored as mutable totals. Every stock movement appends a signed row to `StoreItemHistory` or `RawItemHistory` with a `transaction_type` tag (Purchase, Sale, Production, Blending, Transfer, Return). Any stock level at any point in time can be reconstructed by summing history up to that Unix timestamp, making the system fully auditable.

### Audit Trail

Every transactional table stores `updated_by_id` (FK to `users`) and a Unix timestamp `date_updated`. This gives management a complete record of who changed what and when across purchases, sales, production, and payroll.

---

## Challenges Solved

| Challenge                                                                 | Solution                                                                                                                          |
| ------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------- |
| Per-company data isolation without a heavyweight multi-tenancy framework  | Dynamic database provisioning at company creation; connection routing via central credentials table                               |
| Arabic text in generated PDFs                                             | Custom rendering pipeline combining `arabic-reshaper` + `python-bidi` before passing to ReportLab                                 |
| Real-time stock accuracy across purchase, production, blending, and sales | Immutable `StoreItemHistory` / `RawItemHistory` ledger tables; stock is computed from the ledger, never stored as a mutable total |
| Tiered salesperson incentive calculation                                  | Configurable `calc_incentive` engine with percent-band logic feeding directly into payroll computation                            |
| Supporting both carton-level and unit-level quantities                    | `is_carton` flag + `carton_quantity` stored alongside `quantity`; `units_in_carton` on the product drives conversion              |
| Multi-app Angular build from a single workspace                           | Angular multi-project workspace (`auth`, `setup`) with shared `shared-assets` and `shared-styles` directories                     |

---

## Results & Outcomes

- Eliminated manual spreadsheet reconciliation for inventory, payroll, and sales across all stores
- Gave management a single source of truth for stock levels, supplier balances, and customer outstanding amounts
- Automated monthly payroll with bonus and commission calculations that previously took days to prepare manually
- Delivered 13 distinct PDF report types including bilingual Arabic/English output, directly replacing external reporting tools
- Deployed as a Docker container, enabling repeatable environment provisioning and straightforward cloud deployment

---

## Project Structure

```
.
├── modules/
│   ├── application/        # Auth handlers, user/company management
│   ├── company/
│   │   ├── handlers/       # All business logic handlers (purchases, sales, HR, …)
│   │   └── models.py       # All SQLAlchemy ORM models
│   ├── reports/            # ReportLab PDF generation per report type
│   └── payroll/            # Payroll models
├── common/                 # Database engine, CRUD controller, exceptions, helpers
├── angular/
│   └── projects/
│       ├── auth/    # Login / password reset SPA
│       └── setup/   # Main ERP SPA (all modules)
├── assets/                 # Compiled Angular output, fonts, images
├── resources/              # Seed data (account types, company types, …)
├── Dockerfile
├── requirements.txt
└── setup.py                # Database initialisation script
```

---

## Contact

Built by **Usama Ali** — Full-Stack Developer specialising in Python backends and Angular frontends for business-critical applications.

|          |                                                                    |
| -------- | ------------------------------------------------------------------ |
| GitHub   | [github.com/usamaali012](https://github.com/usamaali012)           |
| LinkedIn | [linkedin.com/in/usamaali012](https://linkedin.com/in/usamaali012) |
| Email    | [usamaali012@gmail.com](mailto:usamaali012@gmail.com)              |

---
