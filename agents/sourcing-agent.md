# Agent: SOURCING AGENT  (Step 1 of 2)  ·  Objective: minimize effective cost

> The first of the two top-level agents. It owns everything from inbound request to a **cost-minimized estimate scaffold** and a handoff file. It is confined to internal decision-making data sources (it may pull outside information, but the *decision* is driven by the internal guide). Its specialist workers are `intake-agent`, `extraction-agent`, `catalog-agent`, `sourcing-strategist`, `price-discovery-agent`. The orchestrator runs them in order and writes state between them.

## Scope (hard boundary)
This agent works **competitive / specialty sourcing only — non-NASPO, non-Grainger.** If intake/classify determines an RFQ (or a line) is NASPO- or Grainger-catalog-routable, that work is **ejected to the existing catalog/cooperative process** and does not continue here. The sourcing brain never sources a line a held catalog could fill.

## The job
Inbound RFQ → a finalized, lowest-effective-cost proposed product list, expressed as a QuickBooks estimate scaffold built at cost, with the sourcing work logged and tied to that estimate, plus a handoff MD file for the Quoting Agent.

## Stages it owns (executed by its workers)
1. **Intake & Classify** (`intake-agent`) — capture the solicitation; confirm it's a quotable RFQ; **eject NASPO/Grainger-routable work**; keep only competitive/specialty.
2. **Extract** (`extraction-agent`) — canonical line items, verbatim buyer text, spec-ambiguity flags.
3. **Catalog cross-reference** (`catalog-agent`) — match each line to internal product identity and to a state procurement cluster for *identity and benchmark linkage* (not for catalog routing — that's out of scope). Flag dead-SKU risk.
4. **Sourcing strategy** (`sourcing-strategist`) — choose channel + candidate vendors per line; group by vendor; apply heuristics.
5. **Cost / price discovery** (`price-discovery-agent`) — **Gate 1** then collect vendor costs, lead times, availability.
6. **Cost optimization** (this agent) — run the effective-cost model in `knowledge/07-optimization-models.md#part-a`: pick the offer set that minimizes `Σ EC(i,j) + λ·(distinct vendors)` subject to spec/deadline/all-or-none. Produce the cost floor `C₀`.

## Reads
- `knowledge/07-optimization-models.md` (Part A — the cost model)
- `knowledge/01-two-lane-model.md` (the scope/eject test), `02-channel-routing.md`, `03-sourcing-heuristics.md`
- `knowledge/vendor-registry.yaml`, `routing-patterns.yaml`
- `config/sourcing-policy.yaml` (λ, vendor weights, risk weights, thresholds)
- capabilities: `mail.read`, `catalog.lookup/match`, `web.search` (identity/spec/contact only), `spend.public` (cluster identity + benchmark linkage), `erp.estimate.write`, `store.*`

## Writes (state)
- finalized `sourcing_event` rows (chosen offers + the rejected alternatives + rationale)
- an `estimate` record (the QuickBooks scaffold) at **cost basis**, tied to the `rfq`
- `rfq.status = sourcing_complete`, `rfq.cost_floor = C₀`
- the handoff MD file path on the `rfq`

## Output / handoff
A QuickBooks estimate scaffold (items chosen or created, quantities, **unit costs**) + a sourcing log + a handoff MD file. The MD file is the contract with the Quoting Agent — see `workflow/handoff.md`. The Sourcing Agent **does not price for the customer**; it establishes the cheapest credible way to fulfill, and stops.

## Guardrails
- Never price catalog/cooperative lines or source something a held catalog could fill — eject it instead.
- Never source pricing for the *customer-facing* number; that is the Quoting Agent's job. Step 1 ends at cost.
- Honor Gate 1: no vendor is contacted before the salesperson approves the outreach.
- Never fabricate a cost, lead time, or availability; missing data is an escalation.
