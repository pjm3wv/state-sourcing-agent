# Optimization Models — Cost Minimization (Sourcing) & Optimal Bid (Quoting)

> The two agents have two different jobs and two different objective functions. The Sourcing Agent **minimizes what it costs us to fulfill**. The Quoting Agent **maximizes the expected value of the bid we submit**. This file defines both formally, names every tunable parameter, and ties each input to the data source it comes from. These are starting formulations meant to be calibrated against real outcomes — see "Calibration & the cold-start problem" at the end.

Scope reminder: this system handles **competitive / specialty (non-NASPO, non-Grainger) sourcing only**. Catalog/cooperative-routable lines are ejected at intake and never reach these models.

---

## Part A — Sourcing Agent objective: minimize effective cost

### The idea
For each RFQ line there are usually several ways to fulfill it — different vendors, different equivalent products, different lead times and reliability. The sticker price is not the real cost. A cheap quote from an unproven vendor with a 6-week lead time on a 2-week deadline is more expensive than it looks. So the Sourcing Agent minimizes **effective cost**, which loads price with risk and rewards preferred vendors, and it prefers to **consolidate** across fewer vendors because every extra supplier adds soft cost (another PO, another shipment, another point of failure).

### Per-offer effective cost
For line *i* and candidate offer *j* (a specific vendor quoting a specific product):

```
EC(i,j) = price(i,j) × qty(i) × (1 + risk(i,j))  −  preference_credit(j)
```

- **price(i,j) × qty(i)** — the quoted extended cost. Source: vendor quote (Price Discovery). Catalog price is never a source here; this lane is sourced.
- **risk(i,j)** — a multiplier ≥ 0 that inflates cost for fulfillment risk:
  ```
  risk(i,j) = w_avail·(1 − availability_conf) + w_lead·lead_penalty(i,j) + w_spec·(1 − spec_fit_conf)
  ```
  - `availability_conf` — dead-SKU risk from the Catalog agent's flag + vendor confirmation.
  - `lead_penalty` — 0 if lead time comfortably beats the deadline; rises as it approaches/exceeds it.
  - `spec_fit_conf` — how sure we are the offered product meets the buyer's spec (equivalents score < exact).
- **preference_credit(j)** — a cost-equivalent reward for vendor quality, from `vendor-registry.yaml`:
  ```
  preference_credit(j) = V(j) × price(i,j) × qty(i)
  ```
  where `V(j)` is a small fraction by tier (e.g. PARTNER 0.06, SOURCING_PARTNER 0.03, TRANSACTIONAL 0, UNKNOWN −0.02). This says "a known partner's quote is worth treating as ~6% cheaper than its sticker because the fulfillment is more certain."

### The portfolio objective (why it isn't just per-line lowest price)
Choosing offers one line at a time ignores consolidation. The agent minimizes total effective cost **plus a penalty for vendor sprawl**:

```
minimize   Σ_i EC(i, j(i))   +   λ_consolidation × (count of distinct vendors used)
   over    the assignment j(i) of one offer to each line
subject to (hard constraints):
   • every chosen offer meets spec and can deliver by the deadline
   • all-or-none: if the solicitation is all-or-none and any line has no feasible offer → flag no-bid, don't partial-source
```

- **λ_consolidation** — the dollar value of removing one supplier from the order. The single most important sourcing knob. High λ → the agent will accept somewhat higher per-line prices to buy from fewer vendors. Low λ → it chases the cheapest line-by-line and tolerates many suppliers.

In practice this is a small assignment problem; a greedy solve (cheapest feasible per line, then merge lines onto shared vendors while the merge saving ≥ the price penalty) is enough at RFQ scale. Exact solve only matters for large mixed baskets.

### What the Sourcing Agent outputs
- The chosen vendor + product + **unit cost** for every line → the **cost floor `C₀`**.
- A QuickBooks **estimate scaffold built at cost** (items chosen/created, quantities, cost basis), tied to the RFQ.
- A **sourcing log** explaining each pick (why this vendor, what was the risk/preference math, what else was quoted) — the audit trail and the training data for tuning.
- The handoff MD file (see `workflow/handoff.md`).

### Tunable parameters (Sourcing)
`λ_consolidation`, the tier weights `V(j)`, the risk weights `w_avail / w_lead / w_spec`, the deadline buffer that defines `lead_penalty`, and the `too_small_to_source` threshold (below which a line is just bought, not sourced). All live in `config/sourcing-policy.yaml`.

---

## Part B — Quoting Agent objective: optimal bid (price-to-win)

### The idea
The Sourcing Agent gave us a cost floor `C₀`. The Quoting Agent decides what to **charge the state**. Bid too high → lose to a competitor. Bid too low → win but leave margin on the table (or fail the company's floor). The right bid maximizes **expected profit = probability of winning × margin if we win**. This is the "game theory" instinct made concrete: we are bidding against a field of competitors whose lowest bid is unknown, so we maximize expected value against the distribution of that lowest competing bid.

### Total internal cost (the floor, loaded with soft cost)
```
C = C₀  +  soft_cost
soft_cost = s_supplier × (number of distinct suppliers in the sourced solution − 1)
          + s_freight  (if freight/handling is on us and material)
```
- `C₀` — the minimized cost from the Sourcing Agent.
- **s_supplier** — the recognized soft cost per extra supplier (coordination, PO admin, multi-shipment receiving). This is the "more suppliers create friction not seen in price" point, applied where it belongs: in the bid floor. (Note it also shaped vendor *selection* via λ in Part A — the two are related but distinct: λ chooses vendors; s_supplier prices the residual friction.)

### Probability of winning as a function of bid
Model `P_win(b)` as a logistic function of how far our bid sits from a competition-adjusted reference price:

```
P_win(b) = 1 / ( 1 + exp( k · ( b_eff − b_adj ) / b_adj ) )
b_eff = b × (1 − pref)        ← our evaluated bid after any SB/DVBE bid preference
```

- **b_adj — the competition-adjusted reference price** (the bid at which win probability ≈ 50%). Derived from the **state procurement data**: the historical winning/paid prices for this product cluster or, if the item is new, its UNSPSC category. Anchor it at a chosen percentile of `CrossSupplier_Pricing` / `Line_Items_Clean` history (e.g. the median paid price, or the observed low if the field looks aggressive).
- **k — competition-intensity steepness**, set by **procurement style** (`acq_method` in the data) and the size of the competitive field:
  - low competition (sole-source-ish, few eligible suppliers, non-competitive F&R) → small k → flat curve → we can price up with little win-prob loss.
  - high competition (open competitive F&R, many SB/DVBE bidders, formal sealed bid) → large k → steep curve → small overages crater win probability → bid near the floor.
  Calibrate k from the **price spread** observed for the cluster: wide spread among past suppliers ⇒ buyers tolerate price variance ⇒ smaller k; tight spread ⇒ larger k.
- **pref — our evaluation preference** (e.g. a 5% SB/DVBE bid preference): the buyer evaluates our bid as `b × (1 − pref)`, so a certified bidder can post a nominally higher price and still win. A direct, legal lever straight into `P_win`.

### The optimal bid
```
b* = argmax_b   P_win(b) · ( b − C )      subject to   b ≤ FR_ceiling
```
- With the logistic `P_win`, `E[π](b) = P_win(b)·(b − C)` is single-peaked; solve numerically (or by the first-order condition). The optimum sits **above cost by a margin that widens as competition weakens (small k) and narrows as it intensifies (large k)**. As k → large, `b*` → just above the floor that still wins; as k → small, `b*` → the reasonableness ceiling.
- **FR_ceiling — the Fair & Reasonable cap.** Even with no competition, "F&R" procurement requires a defensible price. Cap `b*` at the historical benchmark × a tolerance (e.g. ≤ median paid × 1.15, or ≤ historical max). The data gives this ceiling directly. This keeps the bid both optimal *and* compliant.

### What the Quoting Agent shows the human at Gate 2
Not just a number — the tradeoff: **the recommended bid `b*`, its estimated win probability `P_win(b*)`, the expected margin, and the F&R headroom**, plus one or two alternative (b, P_win, margin) points so the salesperson can choose more-aggressive or more-conservative with eyes open. Then it completes the required forms and drafts the customer email → the complete bid package.

### Tunable parameters (Quoting)
`b_adj` anchor percentile, `k` per procurement-style band, the `pref` value(s), `s_supplier` and `s_freight` soft-cost loadings, and the `FR_ceiling` tolerance. All live in `config/sourcing-policy.yaml → margin` and `→ pricing_authority`.

---

## The rigorous version (optional, for later)
`P_win(b) = P(b_eff < min competitor bid)` is exactly the CDF of the **lowest competing bid** (an order statistic). If you later fit a distribution to competitors' bids per category from the procurement data, you can replace the logistic with the empirical complement-CDF and get a defensible, data-driven win curve. The logistic is the smooth, low-data stand-in for that until the field is well characterized.

---

## Calibration & the cold-start problem (read this before trusting the bid model)
The cost model (Part A) runs on data you already have — vendor quotes, lead times, your registry. **The bid model (Part B) needs win/loss outcomes to calibrate `k` and `b_adj`, and that is exactly the data that isn't being captured today** (every RFQ currently logs as "submitted," with no won/lost). Consequences and the path:

1. **Until outcomes exist**, run Part B on the *price benchmark only* — anchor `b_adj` to historical paid prices from the procurement data and use conservative `k` defaults by procurement style. This is already better than a flat markup because it's anchored to what the state actually pays.
2. **Start capturing `rfq_outcome` now** (won / lost / no-bid, and the winning price when discoverable). Every closed bid is a labeled training point.
3. **Then calibrate**: fit `k` and the `b_adj` percentile per category from realized win/loss vs. bid gap. The write-back loop (`workflow/state-machine.md` stage 8) is what feeds this. This is the single highest-leverage data investment for the whole system — the cost side optimizes spend, but the bid side is where margin is won or lost, and it is blind without outcomes.
