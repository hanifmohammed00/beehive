# Example — wells-frogo "Reallocate" feature

First real Beehive run. Lives in the `wells-frogo` repo at `.beehive/`
(`brief.md`, `phase-spec.md`, `wave-plan.md`, `progress.md`), spec of record
`reallocate-spec.md`.

Shape of it:

- **6 phases**, weights `P1:3 P2:3 P3:2 P4:2 P5:8 P6:1` (total 19).
- **hot_files**: `backend/app/main.py`, `backend/app/schemas.py`,
  `frontend/src/components/DiversifyFlow.tsx`.
- **Wave partition**: Wave 1 = P1 (backend) ∥ P2 (frontend) — no file
  overlap. Waves 2–3 = P3 then P4 (both hit all three hot files → can't
  share a wave). Wave 4 = P5. Wave 5 = P6 (docs, alone).
- Only Wave 1 parallelises; the rest is a dependency chain.

Takeaway: parallelism is bounded by hot files, not by how many phases have
their dependencies met. Three "ready" phases that all edit one 3,000-line
file still run one per wave.

This run predates v0.2. What v0.2 would have changed:
- A **Phase 0** landing the shared `schemas.py` request fields + the
  `_credit_cash`/`_debit_cash` signatures, so P3/P4/P5 depend on P0 only.
- P3 + P4 **merged into one phase** (both were serialized purely by the
  `main.py` + `schemas.py` + `DiversifyFlow.tsx` collision).
- Result: P0 alone → {P1, P2, P3+4} in one wave → P5 → P6. ~6 serial units
  drop to ~4, and the middle wave runs 3-wide.
