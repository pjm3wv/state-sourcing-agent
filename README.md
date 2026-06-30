# Sourcing Brain — A Company-Agnostic Framework for Autonomous Government RFQ Sourcing

A reusable "living brain" for sourcing inbound government RFQs with a team of agents, supervised by one salesperson whose job becomes selling — not data entry, not vendor email ping-pong, not cross-referencing catalogs by hand.

This repository is **the brain and its knowledge, not a running application.** It defines the architecture, the agent roles, the data model, the workflow state machine, and the knowledge structure. You wire it into the Claude Agent SDK yourself. Every company-specific fact lives in `config/` and `knowledge/` — the engine itself knows nothing about any one company, certification, contract vehicle, or vendor until you populate it.

---

## The Problem This Solves

A government buyer emails an RFQ. Someone has to: read it, pull the line items, figure out which items are on a catalog you can already price vs. which need a vendor quote, reach the right vendor for each, collect pricing, assemble a compliant quote, and submit before the deadline. Done by hand, a skilled rep can carry maybe a few of these a day. The volume of winnable RFQs is far higher than one person's throughput. **The bottleneck is sourcing labor, not selling skill.** This framework moves the labor to agents and keeps the human on the two decisions that actually need judgment: what to send a vendor, and what to send the customer.

## The Goal (from the whiteboard)

> Find the most efficient vendor or vendors to fulfill an inbound state RFQ — and do it at volume, so the salesperson can focus on selling and on understanding the competitive space.

## What "company-agnostic" means here

The engine is built around capabilities and roles, never around a specific company's facts:

- It does not know your certifications. It reads them from `config/company-profile.yaml`.
- It does not know your contract vehicles, your distributor punchout, or your pricing-authority rules. Those are `config/sourcing-policy.yaml`.
- It does not know your vendors. It reads them from a populated `knowledge/vendor-registry.yaml` built on the schema this repo ships.
- It does not know which agencies behave which way. That accrues in `knowledge/agency-intelligence/` over time.

Drop in two different companies' config + knowledge and the same engine runs two different sourcing operations. That is the point.

---

## Repository Map

Items marked _(remaining)_ are specified in `CLAUDE.md` / `HANDOFF.md` but not yet authored.

```
state-sourcing-agent/
├── README.md                 ← you are here: what it is + how to wire it
├── ARCHITECTURE.md           ← the multi-agent design; read this second
├── CLAUDE.md                 ← design-decision log + build status
├── HANDOFF.md                ← session-to-session handoff (start here if resuming)
├── .mcp.json.example         ← capability→provider bindings; copy to .mcp.json (gitignored)
│
├── config/                   ← COMPANY-SPECIFIC. Fill these in per deployment.
│   ├── company-profile.template.yaml
│   ├── sourcing-policy.template.yaml
│   └── jurisdictions.template.yaml
│
├── agents/                   ← the brain. One file per agent = a system prompt + I/O contract.
│   ├── orchestrator.md                 (owns the pipeline + state + the two gates)
│   ├── sourcing-agent.md               (step 1 — minimize effective cost)
│   ├── quoting-agent.md                (step 2 — optimal bid / price-to-win)
│   ├── intake-agent.md
│   ├── extraction-agent.md
│   ├── catalog-agent.md
│   ├── sourcing-strategist.md
│   ├── price-discovery-agent.md        (remaining)
│   ├── competitive-intel-agent.md      (remaining)
│   ├── quote-assembly-agent.md         (remaining)
│   └── compliance-agent.md             (remaining)
│
├── knowledge/                ← reference the agents read. Schemas ship empty; you populate.
│   ├── 00-procurement-process.md       (general govcon knowledge — remaining)
│   ├── 01-two-lane-model.md            (the scope / eject test — remaining)
│   ├── 02-channel-routing.md           (complexity→channel + sector map — remaining)
│   ├── 03-sourcing-heuristics.md       (the decision rules — remaining)
│   ├── 04-compliance-forms.md          (form bundle + field-map model — remaining)
│   ├── 05-data-hygiene-rules.md        (intake quality rules — remaining)
│   ├── 06-voice-and-outreach.md        (outreach + delivery voice — remaining)
│   ├── 07-optimization-models.md       (the two objective functions — ships usable)
│   ├── market-intelligence.md          (CA procurement schema + products→vendors — remaining)
│   ├── vendor-registry.schema.yaml     (schema + worked example; you populate — remaining)
│   ├── routing-patterns.schema.yaml    (schema + worked example; you populate — remaining)
│   ├── agency-intelligence.schema.md   (per-agency profile schema — remaining)
│   └── agency-intelligence/            (per-agency profiles; grows over time)
│
├── data/                     ← raw datasets (gitignored; schema preserved in data/README.md)
│
├── data-model/
│   ├── ERD.md                ← entities + relationships (remaining)
│   └── schema.sql            ← Postgres/Supabase DDL: Half A + Half B + bridge (remaining)
│
└── workflow/
    ├── state-machine.md      ← the 8-stage lifecycle + transitions (remaining)
    ├── human-gates.md        ← the two human-in-the-loop approval gates (remaining)
    └── handoff.md            ← the Sourcing→Quoting handoff contract (remaining)
```

Read order: `HANDOFF.md` (if resuming) → this file → `ARCHITECTURE.md` → `CLAUDE.md` → `knowledge/07-optimization-models.md` → the `agents/` → fill `config/` → populate `knowledge/` → run `data-model/schema.sql`.

---

## How to wire it into the Claude Agent SDK

This framework is SDK-shaped but SDK-agnostic in expression. Concretely:

1. **Each file in `agents/` becomes one agent's system prompt.** The orchestrator is your top-level agent; the rest are subagents it dispatches. In the Agent SDK, define the orchestrator as the main loop and register the specialists as subagents (or as separate sessions the orchestrator calls as tools).

2. **Each MCP/tool named in `ARCHITECTURE.md` is described by capability, not product.** You bind the capability to whatever you actually run: an ERP connector, a product-database connector, an email connector, a distributor punchout, a web-search tool, a public-spend-data source. The agents reference capabilities (`email.send`, `catalog.lookup`, `punchout.price`) so you can swap providers without touching the brain. Copy `.mcp.json.example` to `.mcp.json` (gitignored) and fill in your real providers + secrets there.

3. **The two human gates are SDK hooks.** Implement them as approval interrupts — the agent pauses, surfaces a draft, waits for the salesperson's yes/edit/no, then continues. See `workflow/human-gates.md`.

4. **State lives in the database, not the context window.** Every RFQ is a row; every line item, sourcing attempt, and quote is a row. Agents read and write state through the data layer so a long-running RFQ survives restarts and so the salesperson can see the whole pipeline. Run `data-model/schema.sql` first.

5. **The knowledge files are retrieval targets.** Agents load the relevant knowledge file(s) at the start of the stage they own. Keep them in the repo (or a vector store) and give each agent read access to the ones its prompt names.

---

## What ships usable vs. what you populate

| Ships usable out of the box | You must populate before first run |
|---|---|
| The agent definitions (the system prompts) | `config/company-profile.yaml` (your identity, certs) |
| `knowledge/07-optimization-models.md` (the two models) | `config/sourcing-policy.yaml` (routing, gates, model params) |
| `knowledge/00–06` (general govcon + decision framework) | `knowledge/vendor-registry.yaml` (your vendors) |
| `.mcp.json.example` (capability bindings to adapt) | `knowledge/routing-patterns.yaml` (your patterns) |
| The data model + SQL | `knowledge/agency-intelligence/` (grows over time) |
| The state machine + gates | `config/jurisdictions.yaml` (the states you sell to) |

The brain is the reusable asset. The config + knowledge is the moat. They are deliberately separate so the first is portable and the second is yours.

---

## Design principles

- **Human owns judgment, agents own labor.** The two gates are non-negotiable: no vendor outreach and no customer-facing quote leaves the system without explicit human approval. Everything between is automated.
- **State is durable and inspectable.** The salesperson can open the pipeline at any moment and see every RFQ's stage, every vendor contacted, every price collected.
- **Knowledge compounds.** Every closed RFQ writes back: which vendor answered fastest for which category and agency, what won, what lost. The next RFQ is faster because the last one finished. The brain gets sharper without code changes.
- **Capabilities, not vendors.** The engine references `punchout.price`, not any one distributor. Swap the binding, keep the brain.
- **Competitive intelligence is public-data-only.** The intel agent works from public award records, public spend-transparency portals, and public solicitation history. It never touches a competitor's private systems.
