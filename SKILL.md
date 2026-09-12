---
name: beehive
description: >
  Run the Beehive Protocol — interview a feature brief into a phase spec,
  partition it into parallel-safe waves, and drive each wave through
  build → integrate → test → independent review → verify, autonomously,
  with a weight-based completion estimate after every wave. Use when the
  user wants to build a multi-phase feature with quality gates, says
  "beehive", "run the hive", "waved build", or points at
  .beehive/phase-spec.md. Also use to resume an in-progress run.
argument-hint: "[intake | plan | build | resume]"
---

# Beehive

Thin wrapper. The method is `PROTOCOL.md` (in this skill directory) — read
it in full, then act as the **Coordinator** role from `ROLES.md`. Platform
mechanics for spawning Builders / Reviewers / Verifiers are in
`ADAPTERS.md`.

Per-project state lives in the target repo's `.beehive/` directory:
`config.yml`, `brief.md`, `phase-spec.md`, `wave-plan.md`, `progress.md`,
`summary.md`, `wave-N/`. If that directory has no `config.yml`, start at intake and
create it — starting points for `config.yml`, `brief.md`, and
`phase-spec.md` are in this skill's `templates/`.

## Dispatch

- **no arg / `resume`** — read the target repo's `.beehive/` state
  (`PROTOCOL.md` §Resuming): newest `wave-*/report.md`, `status`,
  `progress.md`. Continue from there. No `phase-spec.md` → start at intake.
- **`intake`** — run `ROLES.md` §Interviewer against `.beehive/brief.md`
  (create it with the user first if missing). If `.beehive/` still holds a
  completed/abandoned prior run, archive it to `.beehive/archive/<feature>/`
  first. Produces `phase-spec.md` (with a contracts-first Phase 0),
  `config.yml`, and `digest.md`. Ends by setting `mode:` from the "read the
  plan or just build?" question (default `autobuild`).
- **`plan`** — `PROTOCOL.md` §ACT 2: partition `config.yml`'s `spec` into
  waves, write `.beehive/wave-plan.md`.
- **`build`** — `PROTOCOL.md` §ACT 3 wave by wave. `mode: autobuild`
  (default) runs every phase and wave to completion without stopping for
  permission and without ever asking the user to confirm or "activate" the
  next wave — it just starts it — updating `.beehive/progress.md` after each
  wave, stopping only on an unrecoverable block. `mode: review` also stops
  for approval at each gate. A role that crashes gets actively resumed and
  tracked, never silently left; one that never comes back blocks the wave.
  Writes `.beehive/summary.md` on the final wave — token usage per role,
  which economy levers actually fired, and any crash history, across the
  whole run.

## Non-negotiable

- Obey `PROTOCOL.md` §Economy and §Invariant rules — Reviewer and Verifier
  are always fresh subagents, never a Builder; Builders get only their
  phase's scope; test output is pasted raw; two phases sharing a `hot_file`
  never share a wave; a `risk: empirical` phase spikes before it builds;
  fix findings go back to the same Builder warm; the Verifier and
  standard-gate Reviewers run on `models.cheap` — resolve and pass this
  explicitly on every spawn, never rely on it defaulting.
- The Coordinator does not write feature code — only merges, test runs,
  reports, progress, and doc edits.
- Never invoke `config.yml` `heavy_review` — name it in the wave report for
  the user to run.
- In `autobuild`, do not pause the run to ask the user anything except when
  reporting an unrecoverable block. Spawning the next wave's Builders needs
  no sign-off; the wave report is posted and the run continues in the same
  turn.
