# Question 1 Answer

## 1. Issues, Logic Errors, and Data Quality Risks

The draft SQL should not be promoted as a production customer mart. The biggest issue is grain control: it moves between line-item, option, order, customer-location-card, customer-category, and customer-month grains without enforcing a one-row-per-customer contract at the end.

The most material findings are documented in `analysis/q1_sql_review.md`. The top issues are:

- Options are joined on `order_id` only instead of `order_id + lineitem_id`, creating fanout.
- `line_total` becomes null for items without option rows.
- Lifetime spend ignores quantity and option revenue.
- `loyalty_customers` is grouped by customer, card, and location even though the requested output is customer-level.
- The final joins multiply customer rows by joining back to multi-row CTEs on `user_id`.
- `monthly_trends` is joined but not selected, so it only adds cost and duplicate rows.
- Category preference is calculated from non-loyalty data and counts option-expanded rows.
- Recency uses `current_date()` directly, making historical rebuilds non-reproducible.
- Null loyalty `user_id` values are not handled explicitly.
- App/source filtering is inconsistent across CTEs.

Measured evidence from the provided files:

- `order_items.csv`: 203,519 line-item rows.
- `order_item_options.csv`: 193,017 option rows.
- Wrong option join rows: 483,736.
- Correct option join rows: 293,811.
- Extra rows from wrong join: 189,925.
- Wrong/correct join ratio: 1.65x.
- Draft-equivalent final output in a local SQLite benchmark: 55,443 rows.
- Refactored customer mart output in the same benchmark: 5,745 rows.
- Local benchmark: refactored query shape was about 2.9x faster, with median runtimes around 0.89 seconds versus 2.55 seconds on this machine.

The benchmark is directional rather than a Snowflake guarantee. In Snowflake, I would validate using query profile metrics: rows produced at each join, bytes scanned, partitions scanned, spill, warehouse time, and credits.

## 2. Refactored dbt Model Design

I would split the logic into tested dbt models:

- `models/staging/stg_mobile_order_items.sql`
- `models/staging/stg_mobile_order_item_options.sql`
- `models/intermediate/int_alltown_fresh_order_line_items.sql`
- `models/intermediate/int_alltown_fresh_orders.sql`
- `models/marts/mart_loyalty_customer_metrics.sql`

Design choices:

- Staging models standardize names and types but preserve source grain.
- Options are aggregated before joining to line items.
- The line-item model joins on `order_id + lineitem_id`.
- The order model is one row per `order_id`.
- The customer mart is one row per non-null loyalty `user_id`.
- Monthly trends are intentionally excluded from this mart because they are a different grain.
- Recency accepts a dbt variable, `loyalty_metrics_as_of_date`, so historical rebuilds can be reproducible.

## 3. dbt Tests and Documentation

The model YAML is in `models/schema.yml`.

Generic tests included:

- `not_null` on primary identifiers and required metrics.
- `unique` on line-item, order, and customer keys.
- `accepted_values` for controlled fields such as `currency`, `customer_segment`, and `recency_status`.
- `relationships` from option line items back to staged order line items, configured as warning severity because the sample data has a small number of orphaned option rows.

Custom singular tests included:

- `tests/assert_no_order_line_fanout.sql`: proves the line-item model stays one row per `order_id + lineitem_id`.
- `tests/assert_mart_loyalty_customer_metrics_one_row_per_user.sql`: proves the mart stays one row per customer.
- `tests/assert_customer_lifetime_spend_reconciles.sql`: reconciles customer lifetime spend back to order totals.
- `tests/assert_order_item_options_have_matching_line_item.sql`: flags orphaned option records.

Additional production tests I would consider:

- Assert no production app values like `Alltown Fresh - DEVELOPMENT` enter the mart.
- Assert no negative item or option quantities unless returns/refunds are explicitly modeled.
- Assert all loyalty mart rows have `total_orders > 0`.
- Assert `lifetime_spend >= lifetime_item_amount`.
- Add freshness checks on raw mobile ordering sources.
- Add an accepted range or anomaly test for extreme order totals.

## 4. Medallion Layer and Upstream Dependencies

This belongs in the medallion architecture as follows:

- Bronze/raw: source tables or raw file ingestions for `order_items` and `order_item_options`.
- Silver/staging: `stg_mobile_order_items` and `stg_mobile_order_item_options`.
- Silver/intermediate: `int_alltown_fresh_order_line_items` and `int_alltown_fresh_orders`.
- Gold/mart: `mart_loyalty_customer_metrics`.

Upstream dependencies:

- Raw mobile order line items.
- Raw mobile order item options.
- A governed `dim_date` if fiscal calendar or holiday logic is needed.
- Eventually, governed `dim_location`, `dim_customer`, and `dim_product` dimensions if available.

The customer mart should power BI and stakeholder reporting. A separate monthly customer mart should be created if analysts need customer-month trend analysis.
