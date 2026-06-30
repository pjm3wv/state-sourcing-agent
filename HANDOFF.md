# HANDOFF — Sourcing Brain (session-to-session)

> **START HERE.**
> - **Repo:** `pjm3wv/state-sourcing-agent` · **Branch:** `claude/sourcing-brain-setup-ckpf30`
> - **The CA procurement CSVs are gitignored and will NOT be in your clone.** Before any
>   data work, ask the user to re-upload `CA_Procurement_Catalog_1.zip` and extract it into
>   `data/` (the full schema is preserved in `data/README.md`). See §4.

This document hands the **Sourcing Brain** build to a fresh Claude Code session
that starts from a clean clone of this repo. It captures everything that lived
only in the prior working session — current state, an as-built review, the open
decisions, the data situation, and the proposed build plan — so the next session
can take off without that context.

> **Assumed access of the receiving session:** the repo only, plus **public
> data / public web**. It does **not** have the original design-session
> artifacts, the prior chat, or the BBS knowledge bank — anything not in the
> repo must be treated as gone unless re-supplied. Plan accordingly (see §4 and
> §9).

---

## 1. TL;DR — where things stand

- The repo is a **baseline architecture + design framework**, not a running app.
  It defines agents (as prompts), config templates, the optimization models, and
  empty `data-model/` + `workflow/` stubs.
- **Done:** README, ARCHITECTURE, CLAUDE.md, 7 of 11 agent files, 3 config
  templates, and the two optimization models (`knowledge/07-optimization-models.md`).
- **Not built yet:** 4 agent files, all of `knowledge/00–06` + 3 schemas +
  `market-intelligence.md`, the entire `data-model/` (ERD + schema.sql), and the
  entire `workflow/` (state machine, gates, handoff). Full list in §6.
- The CA procurement dataset has been inspected and validated against spec, but
  **its CSVs are gitignored and are NOT in the repo** — see §4.
- **Three decisions are still open** (§5) and **were not confirmed by the user.**
  Recommendations are given; either confirm with the user or proceed on the
  recommended defaults and note that you did.

## 2. Orient yourself (read in this order)

1. `CLAUDE.md` — the design-decision log and build status (authoritative on intent).
2. `README.md` — what the system is and how it wires into the Claude Agent SDK.
3. `ARCHITECTURE.md` — the 8-stage pipeline, the agent topology, the two gates.
4. `knowledge/07-optimization-models.md` — the heart: the two objective functions.
5. The existing `agents/*.md` — the prompt/I-O-contract pattern to imitate.
6. `data/README.md` — the dataset schema (since the CSVs themselves won't be present).
7. This file (§5–§8) — open decisions, review flags, and the build plan.

## 3. Repo state

- **Working branch:** `claude/sourcing-brain-setup-ckpf30` (develop here; do not
  push elsewhere without explicit permission).
- **Baseline commit:** `baseline: sourcing-brain architecture + optimization models`.
- **Committed tree:** `README.md`, `ARCHITECTURE.md`, `CLAUDE.md`, `.gitignore`,
  `agents/{orchestrator, sourcing-agent, quoting-agent, intake-agent,
  extraction-agent, catalog-agent, sourcing-strategist}.md`,
  `config/{company-profile, sourcing-policy, jurisdictions}.template.yaml`,
  `knowledge/07-optimization-models.md`, `data-model/_TODO.md`,
  `workflow/_TODO.md`, `data/README.md`, `data/.gitkeep`.
- `.gitignore` excludes `data/*` (except its README/.gitkeep), `.env*`, keys, and
  `.mcp.json`. **No secrets are committed; keep it that way.**

## 4. The data situation (read before touching the data-engineer track)

The dataset is the **"Aldrich Dynamics — CA Fair & Reasonable" CA procurement
star schema** — the `spend.public` backbone and the price benchmark for the bid
model. In the prior session it was extracted into `data/` and validated:

- Counts matched spec (37,413 line items, 26,774 clusters, 194 suppliers, 589
  buyers, ~$16.4M spend, coverage 06/2024–06/2026).
- Join key `(business_unit + po_number)` confirmed; PO# alone collides.
- Every bid-model input is present in real columns: `acq_method`,
  `unit_price_min/med/max`, `price_spread_pct`, `cert_type`/`certification_type`.
- The products→vendors path is real: `cluster_id` bridges
  Line_Items → Product_Catalog → CrossSupplier_Pricing → Suppliers.

**Critical:** those CSVs are **gitignored by design and will not be in your
clone.** Your `data/` will contain only `README.md` + `.gitkeep`. The full
schema (every column, the join key, loader gotchas, the clustering method) is
preserved in **`data/README.md`** so you can author `data-model/ERD.md` and
`data-model/schema.sql` immediately *without* the files. To actually load/query
data you must first restore the CSVs — **ask the user to re-upload
`CA_Procurement_Catalog_1.zip`** (or regenerate from the FI$Cal public export
per `data/README.md`), then extract into `data/`. Do not assume the data is
present; check `ls data/` first.

## 5. Decisions

**Resolved (applied to the repo in a cleanup pass):**
- **Knowledge-file naming → numbered `00–07`** is the canon; `README.md`'s map
  was corrected to match. Use numbered names for all new knowledge files.
- **Lane-A → detect-and-eject only.** `config/sourcing-policy` now carries
  `two_lane.lane_a.enabled: false` with a scope note; the engine retains Lane-A
  scaffolding but never sources/prices it in this deployment.
- **`agency-intelligence` shape → a per-agency directory** (`knowledge/agency-intelligence/`)
  plus one schema doc (`knowledge/agency-intelligence.schema.md`); the stray
  `intake-agent.md` reference was fixed.

**Still open (your call — not a cleanup item):**
- **What to build now / dispatch.** *Recommended:* launch all four tracks (§8) in
  parallel. Alternatives: hold the data-engineer track until a Supabase project
  is confirmed; or sequence the tracks. This was never greenlit — confirm with
  the user before spawning build subagents.

## 6. Remaining build work (the file inventory)

**`agents/` (4):** `price-discovery-agent.md`, `competitive-intel-agent.md`,
`quote-assembly-agent.md`, `compliance-agent.md`.

**`knowledge/` (7 docs + 3 schemas + 1):** `00-procurement-process.md`,
`01-two-lane-model.md`, `02-channel-routing.md`, `03-sourcing-heuristics.md`,
`04-compliance-forms.md`, `05-data-hygiene-rules.md`, `06-voice-and-outreach.md`,
`vendor-registry.schema.yaml`, `routing-patterns.schema.yaml`,
`agency-intelligence.schema.md`, `market-intelligence.md`.

**`data-model/` (2):** `ERD.md`, `schema.sql` — **Half A** sourcing ops (agency,
rfq, rfq_line_item, product, vendor, vendor_contact, sourcing_event, estimate,
quote, quote_line, compliance_form, field_map, vendor_profile, routing_pattern,
rfq_outcome) + **Half B** market intelligence (proc_line_item, proc_supplier,
proc_buyer, product_cluster, cross_supplier_pricing, brand, unspsc_consistency,
matching the CSV columns so they load directly) + the **bridge**
(product↔product_cluster, vendor↔proc_supplier, agency↔proc_buyer, category↔UNSPSC).

**`workflow/` (3):** `state-machine.md` (8-stage lifecycle + transitions),
`human-gates.md` (the two gates as SDK approval hooks), `handoff.md` (the
Sourcing→Quoting handoff: the MD-file contract + the QuickBooks estimate scaffold
+ the sourcing log).

## 7. As-built review flags

**Fixed in the cleanup pass (verify, then build on them):**
1. ✅ **Knowledge-file naming** — numbered `00–07` canon; README map corrected.
2. ✅ **`agency-intelligence` shape** — per-agency directory + one schema doc;
   `intake-agent.md` reference fixed; `knowledge/agency-intelligence/` created.
5. ✅ **`pricing_authority` default** — `web.search` removed from
   `default_allowed_sources` (cost is vendor-quote/catalog/punchout only; web is
   an identity/benchmark input, never a Part-A cost source).
6. ✅ **`.mcp.json.example`** — added with placeholder capability bindings (no
   secrets); `punchout.price` intentionally left unbound (Lane A out of scope).
7. ✅ **Optimization params seeded** — `config/sourcing-policy → optimization`
   now carries conservative starting defaults for both models (λ, tier credits,
   risk weights, soft costs, `k` bands, `b_adj` percentile, F&R tolerance).
   *Residual:* Part A still doesn't specify how candidate offers per line are
   generated/deduped (vendor count, equivalent-product collapsing) — define this
   when authoring `price-discovery-agent.md` + `03-sourcing-heuristics.md`.

**Partially addressed — finish during the build:**
3. **Lane-A vs. the scope cut.** Config now says detect-and-eject
   (`lane_a.enabled: false`). Still ensure the *knowledge* docs you author
   (`01-two-lane-model.md`, `06-voice-and-outreach.md`) frame Lane A as
   detection-for-ejection, not an active lane, and keep ARCHITECTURE consistent.

**Still open (build work, not cleanup):**
4. **Orchestrator vs. the two top-level agents — who dispatches whom.**
   ARCHITECTURE's diagram shows the orchestrator dispatching the specialists
   directly; CLAUDE.md says it dispatches the two top-level agents (Sourcing,
   Quoting), each owning its workers. Reconcile this in `workflow/state-machine.md`
   + `workflow/handoff.md` — the Step-1→Step-2 boundary and the MD-file handoff
   contract are the most important thing to get right.

## 8. Proposed parallel-subagent build plan

Four tracks write to **disjoint directories**, so they parallelize without file
conflicts. The naming/Lane-A/agency-intel conventions are already locked (§5),
so you can fan out directly.

| Track | Builds | Notes / dependencies |
|---|---|---|
| **A — Agent author** | the 4 remaining `agents/*.md`, imitating the existing 7 | needs naming canon (§5.2) |
| **B — Knowledge author** | `knowledge/00–06`, the 3 schemas, `market-intelligence.md`; BBS facts as clearly-labeled *examples* only | needs naming canon + agency-intel shape (§7.2) |
| **C — Data engineer** | `data-model/ERD.md` + `schema.sql` (Half A + Half B + bridge); validate join key; build the products→vendors query | can author from `data/README.md` now; **loading needs the CSVs restored (§4) + a Supabase project (§9)** |
| **D — Workflow author** | `workflow/{state-machine, human-gates, handoff}.md`; resolves flag §7.4 | needs naming canon |

A, B, D are independent and can run concurrently. C can author files immediately
but cannot load/query data until §4 and §9 are satisfied.

## 9. Environment & access notes

- **Public data is available** to the receiving session; the proprietary CSVs are
  not (see §4). The `competitive-intel` and `market-intelligence` work can use
  public award/spend sources directly.
- **Supabase:** loading the dataset and running `schema.sql` against a live
  project requires a Supabase MCP connector **authorized in an interactive
  session** (`/mcp`). A headless/web session cannot complete OAuth. `schema.sql`
  and `ERD.md` can still be authored as repo files without it.
- **Capability bindings** (`mail.*`, `catalog.*`, `erp.*`, `web.search`,
  `spend.public`/`awards.public`, `store.*`) are referenced by capability, not
  product, and are unbound until a `.mcp.json` is created (see flag §7.6).

## 10. Guardrails (carry forward — non-negotiable)

- **Engine stays company-agnostic.** Company facts (BBS or otherwise) are example
  data in `config/` + `knowledge/` only — never hardcoded in `agents/` or the engine.
- **Lane B only.** Never wire `punchout.price` into the sourcing path; Lane A is
  detect-and-eject.
- **Preserve the two human gates** (Gate 1 vendor outreach, Gate 2 send/submit) in
  any executable wiring.
- **No secrets in the repo.** Keep `.env*`, keys, and `.mcp.json` gitignored.
- **Internal sourcing economics** (exact margin tiers) stay out of any
  shared/exported artifact.

## 11. The calibration / cold-start caveat (highest-leverage open dependency)

The cost model (Part A) runs on data in hand. The **bid model (Part B) needs
win/loss outcomes to calibrate `k` and `b_adj`, and those are not captured
today.** Until they are: anchor `b_adj` to historical paid prices from the CA
data and use conservative `k` defaults by procurement style; and **start
capturing `rfq_outcome` (won/lost/no-bid + winning price when known) from day
one** — every closed bid is a labeled training point. This is the single
highest-leverage data investment in the system.
