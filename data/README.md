# data/ — raw datasets (gitignored)

This folder holds raw source data that is **deliberately excluded from git**
(see `../.gitignore`). Nothing here is versioned except this README and
`.gitkeep`.

## What goes here
The **CA procurement star schema** ("Aldrich Dynamics — CA Fair & Reasonable"),
the `spend.public` backbone and price-benchmark source for the bid model:

| File | Role | Approx. rows |
|---|---|---|
| `Line_Items_Clean.csv` | fact table | 37,413 |
| `Product_Catalog.csv` | product→vendor reverse catalog (clusters) | 26,774 |
| `CrossSupplier_Pricing.csv` | competitiveness / price spread | — |
| `Suppliers.csv` | the competitive set | 194 |
| `Buyers_Depts.csv` | buyers / departments (prospecting) | 589 |
| `Brands.csv` | brand reference | — |
| `UNSPSC_Consistency.csv` | category consistency | — |

Join key across the fact + dimensions: `(business_unit + po_number)`.

## Why gitignored
Raw CSVs are large, may carry redistribution constraints, and belong in the
warehouse (Supabase), not in source control. Drop the CSVs here locally; the
data-engineer workstream loads them and builds `data-model/schema.sql` Half B.
