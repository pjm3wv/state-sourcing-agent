# workflow/ — pending build

This directory will hold the runtime lifecycle docs. **Not yet authored** — see
`/HANDOFF.md §6` and `/CLAUDE.md` for the full spec. To build:

- **`state-machine.md`** — the 8-stage RFQ lifecycle
  (`Intake → Classify → Extract → Enrich+Price → Compliance → Draft → Approve →
  Write-back`) and the legal transitions between stages.
- **`human-gates.md`** — the two human-in-the-loop gates expressed as SDK
  approval hooks: Gate 1 (vendor outreach) and Gate 2 (send/submit).
- **`handoff.md`** — the Sourcing→Quoting handoff contract: the MD-file format,
  the QuickBooks estimate scaffold, and the sourcing log. Also resolve
  `/HANDOFF.md §7.4` here — reconcile whether the orchestrator dispatches the two
  top-level agents (Sourcing, Quoting) or the specialists directly.
