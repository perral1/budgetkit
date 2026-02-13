# BudgetKit

BudgetKit is a PocketBase-backed budgeting toolkit built as standalone HTML modules.

## Modules

- [Budget](budget.html)
- [Core Transaction Register](transaction_register.html)
- [Management (Accounts, Assets, Categories, Payees)](management.html)
- [CSV Importer](importers/txn_csv_importer.html)
- [Budget CSV Importer](importers/budget_csv_importer.html)
- [Collection Bulk Delete Utility](utils/collection_bulk_delete.html)

## Specs

- [Database specification](docs/database_spec.md)
- [Module best practices](docs/module_best_practices.md)

## Notes

- Modules are single-file HTML apps with inline CSS/JS.
- Authentication uses PocketBase `users` auth with bearer tokens.
- Modules are designed to run from `file://`, `http://`, and `https://`.
