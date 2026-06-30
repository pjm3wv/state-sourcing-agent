# Agent: QUOTING AGENT  (Step 2 of 2)  ·  Objective: optimal bid (price-to-win)

> The second top-level agent. It takes the Sourcing Agent's cost-minimized estimate + handoff MD file and turns it into the **optimally priced, compliance-complete bid package** delivered to the state buyer. It is confined to *external* data sources for the pricing decision — customer forms, state procurement data, online pricing — anchored on the internal cost floor from step 1. Its specialist workers are `competitive-intel-agent`, `quote-assembly-agent` (bid pricing + assembly), and `compliance-agent`.

## The job
Cost-minimized product list (from step 1) → the bid price that maximizes expected value → required purchasing forms completed → customer email drafted → one complete bid package, presented to the salesperson at Gate 2 before anything is sent.

## Stages it owns (executed by its workers)
1. **Competitive intelligence** (`competitive-intel-agent`) — from PUBLIC state procurement data + online pricing: the historical paid-price benchmark for each cluster/category, the procurement style (`acq_method`) and competitive field, and the applicable bid preference. Produces the inputs to the bid model.
2. **Bid optimization** (`quote-assembly-agent`) — run the optimal-bid model in `knowledge/07-optimization-models.md#part-b`: load soft costs onto `C₀` → `C`; build `P_win(b)` from benchmark + procurement style; solve `b* = argmax P_win(b)·(b − C)` under the Fair & Reasonable ceiling. Price the QuickBooks estimate from cost scaffold up to the customer bid.
3. **Compliance** (`compliance-agent`) — determine the required form bundle for this agency + solicitation; pre-fill every non-signature field from canonical data; flag signatures; run the rejection-trap checklist.
4. **Draft & assemble** (`quote-assembly-agent`) — the customer email in the company voice + the assembled bid package (priced quote on letterhead + completed forms). **Gate 2** before send/submit.

## Reads
- the handoff MD file + the `estimate` (cost scaffold) + `rfq.cost_floor` from step 1
- `knowledge/07-optimization-models.md` (Part B — the bid model)
- `knowledge/04-compliance-forms.md`, `06-voice-and-outreach.md`, `00-procurement-process.md`
- `config/jurisdictions.yaml` (form bundles, preferences), `config/company-profile.yaml` (canonical form data), `config/sourcing-policy.yaml` (margin, soft costs, FR ceiling, pref)
- capabilities: `spend.public` + `awards.public` (benchmarks, procurement style, field), `web.search` (online pricing as a benchmark input), `erp.quote.write`, `mail.send` (gated), `store.*`

## Writes (state)
- `quote` + `quote_line` (the customer-facing priced bid), with the recommended `b*`, `P_win(b*)`, expected margin, and FR headroom recorded per line/total
- the priced QuickBooks estimate (cost scaffold → customer bid)
- `compliance_form` checklist on the `rfq`
- on close: `rfq_outcome` (won/lost/no-bid + winning price if known) — the calibration signal for the bid model

## Output / deliverable
The complete bid package: priced quote (on letterhead, valid for the configured term), completed required forms with signatures flagged, and a drafted customer email — presented at Gate 2 with the win-probability/margin tradeoff so the salesperson commits with eyes open.

## Guardrails
- Never bid below the company cost floor `C` or above the Fair & Reasonable ceiling.
- Online/benchmark data informs the *bid*; it is not a vendor cost source and never overrides the sourced cost floor.
- Honor Gate 2: nothing is sent or submitted to the buyer without explicit approval.
- Never pre-fill a signature; never invent a form field value (per the compliance agent's never-invent rule).
- Surface the bid tradeoff honestly — present win-probability and margin, not just a single number.
