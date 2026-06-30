# Data Quality Rules

This project treats Data Quality as part of the Analytics Engineering contract.

## Core Checks
- Primary Keys should be unique and not `NULL`.
- Foreign Keys should map to valid Parent records.
- Order-level facts should remain one row per Order.
- Line-level facts should remain one row per Order Line.
- Product Option records should not create unintended many-to-many joins.
- `NULL` Customer identifiers should be flagged before Loyalty Analysis.
- Extreme order values should be flagged for review before Executive Reporting.

## Staging Layer Handling
The staging layer should:
- Standardize column names & data types.
- Preserve source records.
- Add Quality Flags where records are analytically risky.
- Avoid "silently deleting records" without documented Business Logic.
- Prepare clean, typed inputs for Intermediate and Mart Models.
