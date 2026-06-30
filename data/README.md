# data/ — raw datasets (gitignored)

This folder holds raw source data that is **deliberately excluded from git**
(see `../.gitignore`). Nothing here is versioned except this README and
`.gitkeep`. **The CSVs do not travel with the repo** — a fresh clone has an
empty `data/`. This file preserves the full schema so the dataset can be
re-obtained or reloaded without guessing its shape. See `../HANDOFF.md §4`.

## The dataset
**"Aldrich Dynamics — CA Fair & Reasonable Procurement: Cleaned Data & Product
Catalog."** Source: FI$Cal "Detail Information" export (HTML-as-.xls), Fair &
Reasonable purchases. Coverage **06/03/2024 – 06/19/2026**. Built 2026-06-22.

This is the `spend.public` backbone and the price-benchmark source for the
Quoting Agent's bid model (`knowledge/07-optimization-models.md` Part B).

### Key counts (from the dataset README)
- Enriched line items: **37,413** · Total line spend: **$16,431,927**
- Distinct suppliers: **194** · Distinct buyers: **589** · Distinct departments: **63**
- Distinct UNSPSC codes: **3,829**
- Product clusters (catalog rows): **26,774** · Recurring clusters (ordered >1×): **4,197**
- Lines flagged service / non-product: **1,967**

## How to restore the data into this folder
1. **Re-upload** the original `CA_Procurement_Catalog_1.zip` to the session and
   extract here, OR obtain the FI$Cal Fair & Reasonable public export and
   re-run the clustering described under *Method notes* below.
2. Normalize filenames (the zip ships `*-Table 1.csv`; strip the ` -Table 1`
   suffix) so they match the names below.

Expected files after restore:
`Line_Items_Clean.csv`, `Product_Catalog.csv`, `CrossSupplier_Pricing.csv`,
`Suppliers.csv`, `Buyers_Depts.csv`, `Brands.csv`, `UNSPSC_Consistency.csv`,
`README.csv`.

## Files, roles, and columns

### `Line_Items_Clean.csv` — the fact table (source of truth). ~37,412 rows.
Every PO line tied to its supplier + buyer via the composite key.
```
business_unit, department_name, po_number, line_num, item_description, brand,
mpn, mpn_conf, unspsc_code, unspsc_description, mat_material, mat_color,
mat_size, uom, quantity, unit_price, line_total, is_service, cluster_id,
supplier_id, supplier_name, supplier_city, supplier_state, certification_type,
acq_method, buyer_name, buyer_email, po_grand_total, po_start_date, line_status
```

### `Product_Catalog.csv` — one row per product cluster (the catalog / products→vendors reverse index). ~26,773 rows.
```
cluster_id, canonical_description, brand, mpn, mpn_confidence, match_basis,
unspsc_code, unspsc_description, times_ordered, distinct_descriptions,
distinct_suppliers, distinct_buyers, distinct_departments, total_qty,
total_spend, unit_price_min, unit_price_med, unit_price_max, source_suppliers,
sample_desc_variants
```

### `CrossSupplier_Pricing.csv` — clusters sourced by >1 supplier: price by supplier + spread. ~4,405 rows.
The competitiveness view — who is cheapest on the same item. **Prices compared
only within the SAME unit of measure.**
```
cluster_id, match_basis, canonical_description, brand, mpn, uom, supplier_name,
times, qty, avg_unit_price, min_unit_price, max_unit_price, suppliers_same_uom,
price_spread_pct
```

### `Suppliers.csv` — supplier master (= your competitive set). 194 rows.
```
supplier_id, supplier_name, cert_type, city, state, pos, lines, total_spend,
top_categories
```

### `Buyers_Depts.csv` — buyer/department master (prospecting). ~594 rows.
```
buyer_name, buyer_email, department_name, pos, lines, total_spend
```

### `Brands.csv` — detected manufacturers by spend + supplier count. ~92 rows.
```
brand, lines, clusters, suppliers, total_spend
```

### `UNSPSC_Consistency.csv` — UNSPSC codes ranked by classification inconsistency. ~3,828 rows.
High `distinct_desc` = catch-all bucket.
```
unspsc_code, unspsc_description, lines, distinct_desc, distinct_brands,
total_spend, desc_per_line
```

## Method notes (carry into `data-model/schema.sql`)
- **Join key:** `(business_unit + po_number)`. PO# alone collides across
  business units — **never join on PO# alone.**
- **Cluster bridge:** `cluster_id` links `Line_Items_Clean` → `Product_Catalog`
  → `CrossSupplier_Pricing`. The products→vendors query is:
  cluster → `CrossSupplier_Pricing` → `Suppliers`.
- **Loader gotchas:**
  - Money columns are strings with `$` and thousands commas
    (`"$1,387,199.75"`) — strip and cast to numeric on load.
  - `is_service` is a text boolean (`TRUE`/`FALSE`).
  - `source_suppliers` / `sample_desc_variants` are ` | ` and ` || `
    delimited multi-value strings.
- **MPN extraction:** regex on "MANUFACTURER SKU", #codes, P/N, item#.
  Confidence: high = known mfr prefix (MMM=3M, UNV=Universal…), med =
  letters+digits, low = bare number. **Verify low/med before trusting for
  sourcing.**
- **Clustering:** Tier 1 = shared validated MPN. Tier 2 = fuzzy match
  (token_set ≥ 90) on normalized description within the same UNSPSC code.
- **`match_basis` trust label** (per cluster): `MPN-high` / `MPN-med` =
  anchored on a manufacturer part number (treat as same item); `fuzzy-desc` =
  text-only match (review before treating as identical); `singleton` = one-off.
  → Map this to the `spec_fit_conf` term in the Part A risk model.
- **Equivalence caveat:** a cluster is a *likely* same/equivalent product;
  human review is still required before treating two SKUs as substitutable for
  a specific agency (esp. gloves, PPE, medical).
- **Known limits:** brand coverage ~15% (curated dictionary; expandable); the
  state "Item ID" column was empty (no native SKU); 18 lines could not be tied
  to a supplier (orphan lines).
