# data-model/ — pending build

This directory will hold the durable-state schema. **Not yet authored** — see
`/HANDOFF.md §6` and `/CLAUDE.md` for the full spec. To build:

- **`ERD.md`** — entities + relationships.
- **`schema.sql`** — Postgres/Supabase DDL in two halves plus a bridge:
  - **Half A — sourcing operations:** `agency`, `rfq`, `rfq_line_item`,
    `product`, `vendor`, `vendor_contact`, `sourcing_event`, `estimate`,
    `quote`, `quote_line`, `compliance_form`, `field_map`, `vendor_profile`,
    `routing_pattern`, `rfq_outcome`.
  - **Half B — market intelligence:** `proc_line_item`, `proc_supplier`,
    `proc_buyer`, `product_cluster`, `cross_supplier_pricing`, `brand`,
    `unspsc_consistency` — columns mirror the CSVs in `data/` so they load
    directly (see `data/README.md` for every column + the loader gotchas).
  - **Bridge:** `product`↔`product_cluster`, `vendor`↔`proc_supplier`,
    `agency`↔`proc_buyer`, `category`↔UNSPSC.
- Validate the composite join key `(business_unit + po_number)` and build the
  products→vendors query: `cluster_id` → `cross_supplier_pricing` → `proc_supplier`.
