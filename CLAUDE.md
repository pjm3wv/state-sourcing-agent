# CLAUDE.md — Sourcing Brain build context

This is the orienting brain for continuing the **Sourcing Brain** build in Claude Code. It captures every design decision made so far, the current build status, and how to finish — including how to use subagents. Read this first; then read `README.md` and `ARCHITECTURE.md`.

---

## What this project is
A company-agnostic, multi-agent framework for autonomously sourcing **competitive government RFQs** and producing two artifacts: (1) a cost-minimized estimate scaffold, and (2) an optimally priced, compliance-complete bid package. A salesperson supervises; agents do the labor. Built to run on the Claude Agent SDK.

## Locked design decisions (do not relitigate without reason)
1. **Two-step process, two objective functions.**
   - **Sourcing Agent** (step 1) → minimizes *effective cost*. Output: QuickBooks estimate scaffold at cost + sourcing log + handoff MD file.
   - **Quoting Agent** (step 2) → maximizes *expected bid value* (price-to-win). Output: priced bid + completed forms + customer email = complete bid package.
   - The two optimization models are fully specified in `knowledge/07-optimization-models.md`. This is the heart of the system.
2. **Scope cut — Lane B only.** This system handles **non-NASPO, non-Grainger** competitive/specialty sourcing ONLY. NASPO/Grainger-catalog-routable RFQs are *detected and ejected at intake* to the existing Provision Connect / catalog process. The brain never sources what a held catalog can fill.
3. **Supervised autonomy, two human gates.** Runs end-to-end except: **Gate 1** = approve vendor outreach (inside Sourcing Agent); **Gate 2** = approve send/submit (inside Quoting Agent).
4. **Company-agnostic engine + populated knowledge.** The engine references capabilities, never products. All company facts (certs, contract vehicles, vendors, pricing-authority rules, agencies, form bundles) live in `config/` and `knowledge/`. The engine ships empty; the knowledge is the moat. Two companies = two config/knowledge sets, same engine.
5. **State is durable and inspectable** in a database; agents read/write state, not just context. The salesperson can see every RFQ's stage.
6. **The living loop.** On each closed RFQ, write back vendor performance, routing patterns, agency intelligence, and outcome (won/lost/no-bid). The system sharpens without code changes.

## Architecture in one line
Orchestrator owns the pipeline + state + the two gates, and dispatches two top-level agents — **Sourcing Agent** then **Quoting Agent** — each backed by specialist workers.

| Top-level agent | Specialist workers (subagents) |
|---|---|
| Sourcing Agent (step 1, minimize cost) | intake, extraction, catalog, sourcing-strategist, price-discovery |
| Quoting Agent (step 2, optimal bid) | competitive-intel, quote-assembly (bid pricing + assembly), compliance |

## Capability bindings (bind these to real providers in `.mcp.json`)
`mail.read/send` → email/M365 · `catalog.lookup/match` → product DB (Supabase) · `punchout.price` → distributor punchout (OUT OF SCOPE here — Lane A) · `erp.estimate.write` / `erp.quote.write` → QuickBooks · `web.search` → web (identity/spec/contact + online price benchmark only) · `spend.public` / `awards.public` → CA procurement DB · `store.*` → the state database.

## Data sources in hand
- **CA procurement star schema** (the "Aldrich Dynamics — CA Fair & Reasonable" dataset): `Line_Items_Clean` (fact, 37,413 lines), `Product_Catalog` (26,774 clusters — the products→vendors reverse catalog), `CrossSupplier_Pricing` (competitiveness/price-spread), `Suppliers` (194 — the competitive set), `Buyers_Depts` (589 — prospecting), `Brands`, `UNSPSC_Consistency`. Join key: `(business_unit + po_number)`. This is the `spend.public` backbone and the price-benchmark source for the bid model.
- **Vendor registry + routing patterns** (from the BBS knowledge bank): vendor tiers (partner / sourcing-partner / transactional / competitor), category→channel→supplier routing, agency form bundles, voice templates, heuristics. To be expressed as the populated `knowledge/` files.

## BUILD STATUS

### Done
- `README.md`, `ARCHITECTURE.md` (two-step + 8-stage reconciliation + scope cut)
- `config/company-profile.template.yaml`, `config/sourcing-policy.template.yaml` (incl. Two-Lane, pricing-authority, **optimization-model params**, margin, batching, vendor-promotion, seasonal), `config/jurisdictions.template.yaml`
- `agents/`: orchestrator, sourcing-agent, quoting-agent, intake-agent, extraction-agent, catalog-agent, sourcing-strategist
- `knowledge/07-optimization-models.md` (the two objective functions — cost minimization + price-to-win, with tunable params and the calibration/cold-start problem)
- `HANDOFF.md` (session-to-session handoff), `.mcp.json.example` (capability bindings), `data/` + `data/README.md` (gitignored dataset + preserved schema)

### Conventions locked (cleanup pass)
- **Knowledge files use the numbered `00–07` scheme** (README map matches). New knowledge docs follow it.
- **`agency-intelligence`** = per-agency profiles in `knowledge/agency-intelligence/` + one `agency-intelligence.schema.md`.
- **Lane A = detect-and-eject only** in this deployment (`sourcing-policy → two_lane.lane_a.enabled: false`); scaffolding retained for agnosticism.
- **Optimization params live in `config/sourcing-policy.yaml → optimization`** (conservative defaults until calibration).

### Remaining (good first subagent tasks — parallelizable)
- `agents/`: **price-discovery-agent**, **competitive-intel-agent**, **quote-assembly-agent** (bid pricing + assembly), **compliance-agent**
- `knowledge/`: `00-procurement-process.md`, `01-two-lane-model.md` (scope/eject test), `02-channel-routing.md` (complexity→channel + sector map), `03-sourcing-heuristics.md` (decide-lane-first, parallel bid-out, all-or-none, too-small-to-source, target-price, EOFY, etc.), `04-compliance-forms.md` (the vendor_profile + rfq + form + field_map model), `05-data-hygiene-rules.md` (intake noise filtering + ERP dimension intelligence), `06-voice-and-outreach.md` (6-move supplier outreach + buyer "push the leveraged lane" + quote delivery voice), and the schemas `vendor-registry.schema.yaml`, `routing-patterns.schema.yaml`, `agency-intelligence.schema.md`, plus a `market-intelligence.md` describing the CA procurement star schema + the products→vendors method.
- `data-model/`: `ERD.md` and `schema.sql` — TWO halves + the bridge. Half A = sourcing operations (agency, rfq, rfq_line_item, product, vendor, vendor_contact, sourcing_event, estimate, quote, quote_line, compliance_form, field_map, vendor_profile, routing_pattern, rfq_outcome). Half B = market intelligence (proc_line_item, proc_supplier, proc_buyer, product_cluster, cross_supplier_pricing, brand, unspsc_consistency) matching the CSV columns so they load directly. Bridge: product↔product_cluster, vendor↔proc_supplier, agency↔proc_buyer, category↔UNSPSC.
- `workflow/`: `state-machine.md` (the 8-stage lifecycle + transitions), `human-gates.md` (the two gates as SDK approval hooks), `handoff.md` (the Sourcing→Quoting handoff: the MD file contract + the QuickBooks estimate scaffold + the sourcing log).

## How to finish with subagents (suggested dispatch)
- **Subagent A — agent author:** write the four remaining `agents/*.md` from the patterns in the existing seven.
- **Subagent B — knowledge author:** write `knowledge/00–06` + the three schemas + `market-intelligence.md`, generalizing the BBS knowledge bank (keep specifics as clearly-labeled examples).
- **Subagent C — data engineer:** load the CA procurement CSVs into Supabase, build `data-model/schema.sql` (both halves + bridge), validate the join key, and build the products→vendors query (cluster → CrossSupplier_Pricing → Suppliers).
- **Subagent D — workflow author:** write `workflow/` (state machine, gates, handoff).
Run A, B, D in parallel (independent). C is independent too but needs the real CSVs mounted.

## Critical open dependency (flag to the human)
The **bid model (price-to-win) needs win/loss outcomes to calibrate** `k` and `b_adj`. Today RFQs aren't tracked won/lost. Until they are, run the bid model on the price benchmark from the CA procurement data and conservative defaults, and start capturing `rfq_outcome` immediately. This is the single highest-leverage data investment.

## Working rules
- Keep the engine company-agnostic. BBS facts are example data in `knowledge/`, never hardcoded in `agents/` or the engine.
- Never wire `punchout.price` into this system's sourcing path — Lane A is out of scope.
- Honor the two gates in any executable wiring.
- Internal sourcing economics (exact margin tiers) stay out of any shared/exported artifact.
