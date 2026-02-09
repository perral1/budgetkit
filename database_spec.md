# BudgetKit Database Specification (PocketBase)

This file is the authoritative data contract for BudgetKit modules.

## 1. Design Principles

- Single-entry by default: standard budget transactions usually have one entry.
- Multi-entry when needed: transfers, trades, and paystubs can use multiple entries under one transaction.
- No double-entry accounting: transactions are not required to globally balance.
- Envelope budgeting: every dollar of net income is assigned to a budget category.
- Starting balances are represented as normal transactions.
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

### 2.4 `txns`

Transaction header that groups one or more entries.

Fields:
- `ts` (datetime, required)
- `memo` (text, optional)
- `payee` (text, optional)
- `source` (text, optional, e.g. `manual`, `import:schwab`)
- `external_id` (text, optional; import idempotency)
- `meta` (json)

Indexes:
- `ts`
- unique `external_id` (recommended; may include source prefix)

### 2.5 `entries`

Primary economic-effect table.

Fields:
- `txn` (relation -> `txns`, required)
- `ts` (datetime, required; copied from `txns.ts`)
- `account` (relation -> `accounts`, required)
- `category` (relation -> `categories`, optional but recommended)
- `asset` (relation -> `assets`, required; default `USD`)
- `qty` (number, required, signed)
- `memo` (text, optional)
- `status` (select, required): `pending | cleared`
- `meta` (json)

Indexes (critical):
- composite (`account`, `ts`)
- composite (`category`, `ts`)
- composite (`asset`, `ts`)
- composite (`status`, `ts`)
- (`txn`)

### 2.6 `budget_lines`

Monthly envelope assignments by category.

Fields:
- `month` (text, required, format `YYYY-MM`)
- `category` (relation -> `categories`, required)
- `budgeted` (number, required)
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

## 3. Data Semantics (Required)

### 3.1 Normal budget transaction
- One `entries` row.
- `category.kind` is `income` or `expense`.
- `asset` is `USD`.
- `qty` is signed (`+` income, `-` spending).

### 3.2 Transfers
- One `txns` record.
- Two entries.
- Equal and opposite `qty`.
- `category.kind` is `transfer` (or `category` is null).

### 3.3 Trades
- One `txns` record.
- Trade cash leg: `trade_cash` in `USD`.
- Trade asset leg: `trade_asset` in non-`USD` asset.
- Optional fee leg: `fee` with negative `qty`.
- Legs usually reference the same brokerage account.

### 3.4 Paystubs
- One `txns` record.
- Net pay: `income` in checking account, `qty = +NET`.
- Transfers (e.g. 401k) are paired transfer entries.
- Payroll detail rows are stored in a virtual account:
  - Gross: `info`, positive
  - Tax withholdings: `withholding`, negative
  - Deductions: `benefit`, negative
  - Employer-only items: `employer_contrib`

Budget modules must:
- include only `category.kind in {income, expense}`
- exclude `accounts.type = virtual`

## 4. Starting Balances

- Enter all starting balances as normal transactions in the month before the first budgeted month.
- Use a category such as `Starting Balance`.
- This establishes both account balances and initial budget availability.
- No special schema fields are used for starting balances.

## 5. Calculation Reference

For month `M` and category `C`:

- `Income(M)` = sum of `entries.qty` where:
  - `category.kind = income`
  - `status = cleared`
  - `accounts.type != virtual`

- `Activity(M, C)` = sum of `entries.qty` where:
  - `category = C`
  - `category.kind = expense`
  - `status = cleared`

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
