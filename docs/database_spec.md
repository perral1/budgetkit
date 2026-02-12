# BudgetKit Database Specification (PocketBase)

This file is the authoritative data contract for BudgetKit modules.

## 1. Design Principles

- Single-entry by default: standard budget transactions usually have one entry.
- Multi-entry when needed: transfers, trades, and paystubs can use multiple entries under one transaction.
- No double-entry accounting: transactions are not required to globally balance.
- Envelope budgeting: every dollar of net income is assigned to a budget category.
- Starting balances are represented as normal transactions.
- Transaction dates are date-only semantics, stored as datetime normalized to `12:00:00Z`.
- Budgeting is always in the default/base currency.
- Modules should not depend on joins for hot paths; use indexed filters.

## 2. Core Collections

### 2.1 `accounts`

Represents a container where assets live.

Fields:
- `name` (text, required)
- `type` (select, required): `bank | credit | brokerage | cash | crypto | virtual | other`
- `institution` (text, optional)
- `is_archived` (bool, default `false`)
- `meta` (json)

Indexes:
- `type`
- `is_archived`

### 2.2 `assets`

Represents a currency or tradable asset.

Fields:
- `symbol` (text, required, unique) e.g. `USD`, `AAPL`, `BTC`
- `name` (text, optional)
- `precision` (number, optional)
- `meta` (json)

Indexes:
- unique `symbol`

### 2.3 `categories`

Represents the meaning of an entry (budget bucket, transfer, trade leg, payroll detail, etc.).

Fields:
- `name` (text, required)
- `parent` (relation -> `categories`, optional)
- `kind` (select, required)
- `is_archived` (bool, default `false`)
- `meta` (json)

Required `kind` values:
- `income`
- `expense`
- `transfer`
- `trade_cash`
- `trade_asset`
- `fee`
- `withholding`
- `benefit`
- `employer_contrib`
- `info`
- `other`

Indexes:
- `parent`
- `kind`
- `is_archived`

### 2.4 `payees`

Canonical payee directory for transaction headers.

Fields:
- `name` (text, required, unique)
- `card_category` (select, optional): `restaurant | intlrest | grocery | airlines | hotel | rental | parking | toll | taxi`
- `is_archived` (bool, default `false`)

Indexes:
- unique `name`
- `card_category`
- `is_archived`

### 2.5 `txns`

Transaction header that groups one or more entries.

Fields:
- `date` (datetime, required; normalized to `12:00:00Z`)
- `memo` (text, optional)
- `payee` (relation -> `payees`, optional)
- `source` (text, optional, e.g. `manual`, `import:schwab`)
- `external_id` (text, optional; import idempotency)
- `meta` (json)

Indexes:
- `date`
- unique `external_id` (recommended; may include source prefix)

### 2.6 `entries`

Primary economic-effect table.

Fields:
- `txn` (relation -> `txns`, required)
- `date` (datetime, required; copied from `txns.date`, normalized to `12:00:00Z`)
- `account` (relation -> `accounts`, required)
- `category` (relation -> `categories`, optional but recommended)
- `asset` (relation -> `assets`, required; default `USD`)
- `qty` (number, required, signed fixed-precision integer)
- `memo` (text, optional)
- `status` (select, required): `pending | cleared`
- `meta` (json)

Indexes (critical):
- composite (`account`, `date`)
- composite (`category`, `date`)
- composite (`asset`, `date`)
- composite (`status`, `date`)
- (`txn`)

### 2.7 `budget_lines`

Monthly envelope assignments by category.

Fields:
- `month` (text, required, format `YYYY-MM`)
- `category` (relation -> `categories`, required)
- `budgeted` (number, required fixed-precision integer)
- `note` (text, optional)
- `meta` (json)

Constraints and indexes:
- unique (`month`, `category`)
- index `month`
- composite (`category`, `month`)

Notes:
- Budgeting is always in the default/base currency.
- There is no separate budget-period table.
- Carryover is derived from history and not persisted.

### 2.8 `v_activity_by_category_month` (view)

Server-side monthly activity aggregation by category name.

Required fields:
- `id` (text, unique per category/month)
- `category` (text; category name from `categories.name`)
- `month` (text, format `YYYY-MM`)
- `activity` (number; summed `entries.qty` fixed-precision integers)

Reference view query:
- Uses `entries` joined to `categories`
- Includes `category.kind in {income, expense}`
- Groups by `strftime('%Y-%m', e.date), c.name`

## 3. Data Semantics (Required)

### 3.0 Fixed-Precision Amount Storage
- All persisted amounts are stored as signed integers in minor units.
- Scale is determined by `assets.precision` for the related asset.
- If `assets.precision` is null/blank/invalid, modules must default to precision `2`.
- Inputs must not exceed the asset precision. Example: for `USD` precision `2`, `10.51` is valid and stored as `1051`, while `10.511` is invalid.
- Modules must reject invalid amount input (manual entry/import) rather than silently rounding.
- Display values must convert persisted minor units back to major units using the same precision.

### 3.1 Normal budget transaction
- One `entries` row.
- `category.kind` is `income` or `expense`.
- `asset` is `USD`.
- `qty` is signed (`+` income, `-` spending) and stored in minor units.

### 3.2 Transfers
- One `txns` record.
- Two entries.
- Equal and opposite `qty` (minor units).
- `category.kind` is `transfer` (or `category` is null).

### 3.3 Trades
- One `txns` record.
- Trade cash leg: `trade_cash` in `USD`.
- Trade asset leg: `trade_asset` in non-`USD` asset.
- Optional fee leg: `fee` with negative `qty` (minor units).
- Legs usually reference the same brokerage account.

### 3.4 Paystubs
- One `txns` record.
- Net pay: `income` in checking account, `qty = +NET` (minor units).
- Transfers (e.g. 401k) are paired transfer entries.
- Payroll detail rows are stored in a virtual account:
  - Gross: `info`, positive
  - Tax withholdings: `withholding`, negative
  - Deductions: `benefit`, negative
  - Employer-only items: `employer_contrib`

Budget modules must:
- include only `category.kind in {income, expense}`
- consume monthly activity from `v_activity_by_category_month` for activity calculations

### 3.5 Archival Semantics (`is_archived`)
- Applies to collections that define `is_archived` (`accounts`, `categories`, `payees`).
- `is_archived = true` means the record is not a selectable option for new search/picker flows and not offered as a default choice when editing transactions/entries.
- Existing references remain valid:
  - `entries.account`, `entries.category`, and `txns.payee` may point to archived records.
  - Modules must still resolve and display those linked archived records correctly.
  - Archived-linked transactions/entries must remain included in all calculations, balances, and totals.
- List-style modules for archived-capable collections should support filtering by `unarchived`, `archived`, or `both`.
- Budget grid behavior for archived categories:
  - The UI may hide archived category line items.
  - Hidden archived categories must still contribute to group totals and overall totals.
  - Activity and availability math must include archived categories when referenced by data.

## 4. Starting Balances

- Enter all starting balances as normal transactions in the month before the first budgeted month.
- Use a category such as `Starting Balance`.
- This establishes both account balances and initial budget availability.
- No special schema fields are used for starting balances.

## 5. Calculation Reference

For month `M` and category `C`:

- `Activity(M, C)` = `v_activity_by_category_month.activity` where:
  - `month = M`
  - `category = C.name`
  - default to `0` if no row exists

- `Available(M, C)` = `Available(M-1, C) + budgeted(M, C) + Activity(M, C)`

## 6. Interop Rules

- Modules must not assume PocketBase system fields like `created` or `updated` are query-safe.
- Any field referenced in filters or sorting must be known to exist in the target collection.

## 7. Non-Goals

- No global balancing enforcement.
- No persisted net-worth or base-currency valuation.
- No automatic lot/cost-basis tracking.
- No implicit category inference.
- No server-side invariants for these semantics; modules enforce behavior.
