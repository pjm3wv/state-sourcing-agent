# ARCHITECTURE — The Multi-Agent Sourcing Brain

The centerpiece. It reconciles three inputs into one system:

- the **whiteboard** ("Sourcing Agent Data Model") — the data-flow skeleton
- the **8-stage pipeline** from the partner package — the fixed, portable backbone
- the **multi-agent execution model** — how the work actually gets done within those stages

Read `workflow/state-machine.md` and `knowledge/01-two-lane-model.md` alongside this.

---

## The thesis

**The pipeline is portable. The jurisdiction + relationship knowledge is the moat.** The eight stages below are identical for every company and every customer. What changes per deployment is the *knowledge* the stages read — certifications, contract vehicles, the lane rules, the channel map, the vendor relationships, the agency form bundles. The engine ships empty; the knowledge makes it yours. This is why the same brain can run two different distributors' sourcing operations from two different config + knowledge sets.

---

## The fixed 8-stage pipeline (the backbone)

```
INBOUND ─▶ 1 INTAKE ─▶ 2 CLASSIFY ─▶ 3 EXTRACT ─▶ 4 ENRICH+PRICE ─▶ 5 COMPLIANCE ─▶ 6 DRAFT ─▶ 7 APPROVE ─▶ 8 WRITE-BACK ─▶ CRM/ERP
                                                          │                                            ▲
                                                   (Gate 1 here)                               (Gate 2 here)
```

| # | Stage | What happens |
|---|---|---|
| 1 | **Intake** | Capture the solicitation from any channel; create the RFQ record. |
| 2 | **Classify** | Two decisions: (a) is this a quotable RFQ vs. tracking/ETA/general noise? (b) which **lane** — leveraged-catalog or competitive-sourcing? |
| 3 | **Extract** | Pull line items, quantities, agency, due date, solicitation number, ship-to. |
| 4 | **Enrich + Price** | Match items to catalog, route each line to a channel, source pricing. **Gate 1** (approve vendor outreach) lives inside this stage. |
| 5 | **Compliance** | Determine required forms/certs for this jurisdiction + solicitation; pre-fill from canonical data; flag signatures. |
| 6 | **Draft** | Assemble the customer quote + acknowledgment in the company's voice; build the ERP import. |
| 7 | **Approve** | **Gate 2** — the salesperson reviews the full quote, margin, and compliance packet, then sends/submits. |
| 8 | **Write-back** | Log the opportunity and (on close) the outcome to the system of record; update the living knowledge. |

Approval is deliberate, not a limitation. The agent prepares; a person commits. That keeps the system deployable into a regulated procurement workflow on day one.

---

## How the agents execute the pipeline

The whiteboard drew one linear agent. At volume, a single agent's context fills with intake parsing, catalog data, vendor threads, and pricing math at once, and it degrades. So the work is split: an **orchestrator** owns the pipeline and state; **specialist subagents** each own a stage (or part of one). The mapping:

| Pipeline stage | Agent(s) that execute it |
|---|---|
| 1 Intake | **Intake Agent** |
| 2 Classify | **Intake Agent** (quotable? + which lane?) |
| 3 Extract | **Extraction Agent** |
| 4 Enrich + Price | **Catalog Agent** → **Sourcing Strategist** → **Price Discovery Agent**, with **Competitive Intel Agent** in parallel |
| 5 Compliance | **Compliance Agent** |
| 6 Draft | **Quote Assembly Agent** |
| 7 Approve | Human gates, enforced by the **Orchestrator** |
| 8 Write-back | **Orchestrator** learning loop |

---

## System shape

```
                          ┌─────────────────────────────┐
   Inbound RFQ            │        ORCHESTRATOR          │
   (email / portal) ─────▶│   owns 8-stage pipeline      │
                          │   + durable state            │
                          │   + the two gates            │
                          └───────────────┬─────────────┘
                                          │ dispatches one stage at a time
   ┌──────────┬───────────┬───────────────┼────────────┬──────────────┬─────────────┐
   ▼          ▼           ▼               ▼            ▼              ▼             ▼
┌────────┐┌──────────┐┌─────────┐┌────────────────┐┌────────────┐┌──────────┐┌────────────┐
│ INTAKE ││EXTRACTION││ CATALOG ││   SOURCING     ││   PRICE    ││COMPLIANCE││   QUOTE    │
│        ││          ││         ││   STRATEGIST   ││ DISCOVERY  ││          ││  ASSEMBLY  │
└────────┘└──────────┘└─────────┘└────────────────┘└────────────┘└──────────┘└────────────┘
                                          │              │
                                          │      ┌───────┴────────┐
                                          │      ▼                ▼
                                          │  (vendor email)  ┌──────────────┐
                                          │                  │ COMPETITIVE  │
                                          │                  │    INTEL     │
                                          │                  └──────────────┘
                                          ▼
   ════════════════════════════ DURABLE STATE (database) ════════════════════════════
     agency · rfq · rfq_line_item · product · vendor · vendor_contact · sourcing_event
     · quote · quote_line · compliance_form · field_map · vendor_profile · rfq_outcome

        ▲                                                          ▲
        │  GATE 1: approve vendor outreach        GATE 2: approve send / submit  │
        └──────────────────────  SALESPERSON  ──────────────────────────────────┘
```

---

## The pivotal logic: the Two-Lane Decision

Almost every RFQ resolves into one of two lanes, and **deciding the lane first is the single most important step** — it determines whether you compete at all. This is generalized in `knowledge/01-two-lane-model.md`; the short version:

- **Lane A — Leveraged Catalog Lane.** The requested items are available on a contract/catalog the company already holds (a cooperative or statewide agreement), or the buyer cites a catalog/contract item number. Mechanism: quote on that vehicle; it requires **only one bid**, so the agency avoids a formal competitive solicitation and gets faster delivery. **Reps proactively push eligible buyers into this lane.** Pricing comes only from the authorized source for that vehicle (e.g., a distributor punchout) — never from open web search.
- **Lane B — Competitive / Specialty Lane.** Items are not on a held catalog (specialty fire, tactical, medical, food, furniture, custom), or the solicitation is an explicit sealed bid. Mechanism: multi-vendor sourcing (the Sourcing Strategist + Price Discovery), then assemble the full compliance packet.

The lane is *pre-classified* at stage 2 and *finalized* per-line at stage 4 (some RFQs are mixed — most lines Lane A, a few Lane B). The boundary is exact-or-equivalent catalog availability; if the catalog can't match, the line falls to Lane B.

---

## The specialists

Each is a single-responsibility subagent. Full prompts in `agents/`.

| Agent | Owns | Reads (knowledge) | Writes (state) |
|---|---|---|---|
| **Intake** | Detect that an inbound message is a quotable RFQ; filter portal/ETA/tracking noise; extract header; pre-classify the lane | `05-data-hygiene-rules.md`, `agency-intelligence` | `rfq` (header), `agency` |
| **Extraction** | Turn RFQ documents (PDF / spreadsheet / email body) into canonical line items | — | `rfq_line_item` |
| **Catalog** | Match each line to the product catalog; tag Lane-A-eligible (catalog/equivalent found) vs. needs-sourcing | catalog via `catalog.lookup` | `rfq_line_item.catalog_match`, `lane` |
| **Sourcing Strategist** | For Lane-B lines, pick the channel per the channel-routing map; group lines by vendor; apply heuristics | `01-two-lane-model.md`, `02-channel-routing.md`, `03-sourcing-heuristics.md`, `vendor-registry`, `routing-patterns` | `sourcing_event` (planned) |
| **Price Discovery** | Execute the plan: draft vendor outreach in the company voice, run allowed research, collect + normalize responses + lead times | `06-voice-and-outreach.md`, `vendor-registry` | `sourcing_event` (requested/responded), unit costs |
| **Competitive Intel** | Pricing/positioning context from PUBLIC data: prior awards, spend history, who else bids | `00-procurement-process.md`, public spend + award sources | `spend_history`, intel notes on `rfq` |
| **Quote Assembly** | Apply margin policy; build the customer quote + ERP import; check quote-validity vs. supplier-quote expiry | `sourcing-policy.yaml` (margin rules), `06-voice-and-outreach.md` | `quote`, `quote_line` |
| **Compliance** | Determine the required form bundle for this jurisdiction + solicitation type; pre-fill every non-signature field from canonical data; flag signatures; run the rejection-trap checklist | `04-compliance-forms.md`, `jurisdictions.yaml`, `company-profile.yaml` | `compliance_form` checklist on `rfq` |

---

## Capability bindings (the "MCP" layer, generalized)

Agents never name a product. They call capabilities. You bind each to whatever you run.

| Capability | What it does | Bind to (examples — your choice) |
|---|---|---|
| `mail.read` / `mail.send` | Read inbound, send outbound (gated) | Any email/MCP connector |
| `catalog.lookup` / `catalog.match` | Query + fuzzy-match the internal product catalog | Your product DB / data warehouse + embedding match |
| `punchout.price` | Account-specific pricing on a held catalog/cooperative vehicle | Your distributor punchout |
| `erp.quote.write` | Push a quote into the system of record | Your ERP / quoting system |
| `web.search` | Open-web research for product identity / spec / the real government-sales contact | Any web search tool |
| `spend.public` / `awards.public` | Public spend-transparency + award/solicitation history | Public procurement record sources |
| `store.read` / `store.write` | Durable RFQ/line/quote/compliance state | Your database (see `data-model/schema.sql`) |

**Pricing-authority rule (config, not hardcoded).** Some companies require that pricing for a given lane/vehicle come *only* from an authorized source (e.g., a distributor punchout) and *never* from open web search. The engine does not assume this — it reads a `pricing_authority` map from `config/sourcing-policy.yaml`, and the Price Discovery agent obeys it. A different company writes a different map. (In the worked example, catalog/cooperative pricing is punchout-only; web search is allowed for product *identity* and *contact discovery*, never for catalog price.)

---

## Why these boundaries

- **Intake ≠ Extraction.** "Is this even a quotable RFQ, and which lane?" is classification; parsing line items is structured extraction. Different failure modes, different prompts. Classification also catches the noise that pollutes a CRM — automated portal notices, ETA chases, tracking pings (see data-hygiene rules).
- **Catalog ≠ Sourcing Strategist.** "What is this item and do we already carry it on a held vehicle?" (catalog + Lane-A test) is matching; "given it's Lane B, which door do we knock on?" is routing that depends on relationships and the channel map.
- **Price Discovery ≠ Competitive Intel.** One gets *your cost* (vendor-facing, fast). The other reads *the market* (public-data-facing, can be slower). Splitting them keeps the cost path quick and lets intel run in parallel without blocking the quote.
- **Compliance is its own agent.** The required forms vary by jurisdiction and solicitation type, and one missing/blank form disqualifies an otherwise-winning bid. It deserves a dedicated check with a rejection-trap checklist, not a footnote in assembly.

---

## The two gates (supervised autonomy)

End-to-end autonomous except for two hard stops, both owned by the orchestrator and detailed in `workflow/human-gates.md`:

- **Gate 1 — Vendor outreach** (inside stage 4). Before any message goes to a vendor, the salesperson sees the drafted outreach (recipient, items, deadline) and approves / edits / cancels. Protects vendor relationships; catches mis-routes and dead-SKU risk.
- **Gate 2 — Send / submit** (stage 7). Before the customer quote is sent or the bid submitted, the salesperson sees the full quote, margin, compliance checklist, and validity dates and approves / edits / cancels. The revenue-and-reputation gate.

Everything between — intake, classify, extract, catalog, strategy, price collection, intel, compliance pre-fill, draft — is autonomous. The salesperson's day becomes: review two well-prepared decisions per RFQ, and sell.

---

## Why a catalog lookup isn't enough (why the human stays)

A lookup returns a list price. It does not tell you whether the SKU still exists, who actually quotes government-distributor business, or what the real number is once a relationship is in play. Six gaps keep a human in the loop (generalized in `03-sourcing-heuristics.md`):

1. **Subject lines lie** — a "competitive" RFQ is often catalog-routable, and vice versa; triage takes judgment.
2. **SKUs go dead** — manufacturers discontinue quietly; only a person at the vendor confirms availability. A dead-SKU "win" is a fulfillment failure.
3. **Specs are ambiguous** — "heavy-duty, black, medium" resolves to a real product only through a conversation.
4. **Price is relational** — the best number comes from a rep who knows the account and the volume, not a portal.
5. **The right contact is hidden** — "who quotes government distributor business?" is rarely the address on the website.
6. **Freight + timeline are real** — FOB-destination pricing and lead time decide whether a bid is even winnable.

Division of labor: **the agent owns breadth and speed** (triage, multi-vendor outreach, follow-up, paperwork); **the human owns depth and relationship** (the call that confirms the SKU, resolves the spec, moves the price). The system scales the rep; it does not pretend to be one.

---

## How the brain gets smarter (the "living" part)

On every closed RFQ, the orchestrator triggers a write-back:

- **Vendor performance** → `vendor-registry`: response time, did they answer, how competitive, for which category + agency. A transactional supplier that answers fast and competitively three times for the same category gets promoted toward a recurring sourcing partner.
- **Routing patterns** → `routing-patterns`: "agency X + category Y → vendor Z answered first and won." Next time that pattern appears, the Sourcing Strategist proposes Z immediately.
- **Agency intelligence** → `agency-intelligence`: how the buyer behaves (preferred lane, form bundle, decision drivers, the real contacts vs. portal noise).
- **Outcome** → `rfq_outcome`: won / lost / no-bid, competitor if known, lesson. This is the win/loss memory most CRMs never capture — and the one the partner package calls out as the measurable win: solicitations stop dying in an inbox, and every inbound becomes a recorded, reportable opportunity.

No code changes. The knowledge files and tables fill in, and the same engine makes better calls.
