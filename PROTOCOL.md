# Beehive Protocol v0.7

<!-- v0.7: token-cost findings from a second real run (wells-frogo
     "future-features", mid-run: 5 waves, 11 phases, repeated spend-limit
     hits). `digest.md` refresh now compacts instead of only appending — a
     live run's digest had grown to 234 lines against its own ~150-line
     budget by wave 2, entirely from per-wave "what changed" sections never
     folded back in, and every role re-reads the whole file every wave.
     Reviewer is now scoped to its own wave's phases in `phase-spec.md`, not
     the whole file — matching what the Verifier's prompt already said — since
     an 11-phase, 615-line spec was being read in full by a Reviewer checking
     a 3-phase wave. §ACT 3 step 1 said "spawn all of the wave's Builders at
     once", directly contradicting §Speed lever 10's own staggering cap; the
     run hit exactly this in wave 3 (3 builders plus a still-live Reviewer
     crashed together on one spend-limit hit), so step 1 now spawns up to the
     cap and backfills, and lever 10 now says a limit-triggered crash is a
     signal to cut the cap further for the rest of the run, not just resume
     the same width. §Speed lever 4 (risk-proportional gates) now covers
     *when* a gate runs, not just how deep: a leaf phase (nothing later
     `depends_on` it) can have its Review+Verify deferred and folded into a
     later wave's gate, guarded so it can't be forgotten — `wave-plan.md`
     records it, the combined gate's diff must span back to cover it, and
     `100% · complete` can't be reported with one still outstanding. This
     formalizes a call the human made by hand mid-run (defer the leaf
     frontend wave's gate into the final wave; keep every backend/contract
     wave's gate immediate, since bugs in those compound into what's built on
     them next) after checking it against the same rework-compounding
     argument §Speed already makes — a phase anything later depends on still
     gates at its own wave, unchanged. §Economy gets one rule that ponytail
     itself doesn't have: minimal in-code comments in the Builder's diff, not
     just a bounded prose report around it — a verbose diff gets paid for
     again by every Reviewer, Verifier, and later Builder who reads it.
     §Speed lever 9 (model tiering) now routes by stakes, not by role name:
     the same run reserved its strong model for full-gate Reviewers and
     `risk: empirical` Builders specifically — where a wrong call is
     expensive to discover late — and ran every routine Builder on the cheap
     tier instead of strong-by-default, with no drop in what the gate caught
     (the fresh Reviewer still catches what a cheaper Builder misses; that's
     the isolation guarantee doing its job regardless of which tier wrote the
     diff). Still two tiers, just re-routed; this is a cost lever, not a
     token-count one, same as the rest of lever 9. Five more findings from
     the same run's post-mortem: (1) a fix round can cost nearly as much as
     the phase's original build — §ACT 3 step 1 now says that's a signal the
     resume didn't actually stay warm, not that fixes are inherently
     expensive, and to check the per-role `tokens` field rather than assume.
     (2) The run's single worst dispatch mistake — two parallel Builders
     inheriting the Coordinator's own ambient cwd, ~200k tokens lost — wasn't
     caught until both agents were already confused; `ADAPTERS.md` now
     requires confirming worktree paths are distinct *before* a batched
     dispatch, not after. (3) Lever 10's crash-cut-the-cap rule now adds:
     when crashes cluster at spawn/first-resume rather than spread through a
     wave (this run: 10 of 14), the trigger is spawn *simultaneity*, not
     sustained width — stagger the dispatches themselves. (4) §Report's
     `tokens` field can be actively wrong, not just absent: a resume call
     that reports only its own (small) cost after the real work finished
     pre-crash looks like a cheap pass that was actually expensive elsewhere
     — flag it rather than fold it in silently. (5) The Coordinator writing
     small glue/fix code itself, instead of spawning a Builder, measured at
     zero marginal subagent cost for exactly that pattern in this run — now
     a named, narrow exception in §Roles, guarded by "a gate still has to
     cover it." Beehive is also Claude Code-only as of this version —
     `ADAPTERS.md` dropped its Codex/Cursor/Aider/manual sections and the
     platform-neutral framing came out of this file, `ROLES.md`, and
     `README.md`.
     v0.6: §Report — each wave report gets a best-effort `tokens` field (per
     role) and a `mechanisms` checklist recording which v0.5 levers actually
     fired this wave (spike ran, cheap model used, tests.log reused,
     checkpointed, concurrency staggered) instead of just what the spec
     says should happen. The Coordinator writes `.beehive/summary.md` on the
     final wave, aggregating both across the whole run — one file to check
     whether a run did what the protocol claims and where its tokens went.
     Crash/resume is now tracked, not assumed: a `resume` field records each
     role's crash→resume cycles, and a role that crashed and never came back
     is `stuck` and blocks `status: passed` — from a run where 4 of 11 role
     instances went permanently unresponsive after a crash and nobody
     noticed until reading a hand-built table after the fact (§ACT 3 step 1,
     §Report, `ADAPTERS.md`). Fixed a real bug found in that same run: the
     model-tiering rule told the Coordinator to route a "light-gate
     Reviewer" to `models.cheap`, but a `light` gate spawns no separate
     Reviewer agent at all (§ACT 3 step 4) — so that instance never existed
     to match against, and the one real cheap-eligible Reviewer (the
     `standard`-gate one) silently never got tiered. Renamed it
     `standard-gate Reviewer` everywhere and required the model be resolved
     and passed explicitly on every spawn, not left to default.
     v0.5: token-cost hardening from a large real run (~5M subagent tokens).
     risk: empirical phases get a cheap spike before the full build, so a
     wrong approach is caught at spec time not verify time. Model tiering is
     mandatory, not advisory — the Interviewer sets models.cheap. Reviewer
     reads tests.log instead of re-running the suite; Verifier narrows to
     end-to-end. Fix findings go back to the same Builder warm, and the
     recheck gate is derived from the fix diff, not the phase. Reviewer /
     Verifier checkpoint their output as they go so a rate-limited one
     resumes. Coordinator may drop context between waves. Concurrency is
     staggered on metered plans.
     v0.4: autobuild is fully unattended — the Coordinator never asks the
     user to confirm, approve, or "activate" a wave; wave activation is
     automatic and the only halt is an unrecoverable block. "Waiting" in
     autobuild always means waiting on another agent, never on the user.
     v0.3: first real run (wells-frogo diversify-search, examples/). Branch
     names deconflicted (wave-N-int); Phase 0 gate depth by whether it lands
     behaviour; resume-a-dead-builder rule; deferrals must be tracked;
     archive a prior run; Verifier seeds its own data. -->

A method, run as a Claude Code skill, for taking a feature from a messy
brain-dump to merged code: interview the human into a spec, partition the
spec into waves that build in parallel where safe, and gate each wave behind
an independent review and a real end-to-end verification.

Speed comes from **breaking dependencies so more work is parallel** — an
interface-first Phase 0, risk-proportional gates, and pipelined
review/verify — not from throwing more builders at a chain. See §Speed.

The Coordinator role (`ROLES.md`) runs this from a Claude Code session,
spawning Builders/Reviewers/Verifiers as subagents (`ADAPTERS.md` for the
mechanics). All state lives in `.beehive/`; a Coordinator session resumes a
run by reading it, even after dropping its own context.

Install this directory as the Claude Code skill (`README.md`). The target
repo supplies `config.yml` and `brief.md`.

---

## The three acts

```
ACT 1 — INTAKE   human writes brief.md → Interviewer explores + questions →
                 phase-spec.md (Phase 0 = shared contracts) + config.yml
                 + digest.md → human approves
ACT 2 — PLAN     Coordinator partitions phases → wave-plan.md → human approves
ACT 3 — BUILD    per wave: build ∥ → integrate → test → review/verify (depth
                 by gate) → merge; next wave's build overlaps this one's gates
```

At the end of Act 1 the Interviewer asks one last question — **"read the
plan first, or just build?"** — which sets the autonomy mode for Acts 2–3:

| mode | Act 2/3 gates |
|---|---|
| `autobuild` (default) | Coordinator runs every wave start to finish without stopping between phases or waves. Wave activation is automatic — it does **not** ask the user to confirm, approve, or "start" a wave at any point. It writes a report after each wave and updates progress (§Progress), but never waits for a reply. It stops **only** on an unrecoverable block: tests still red after builder retries, a verify fail, or a merge conflict that exposes a spec bug. |
| `review` | Coordinator additionally stops for human approval after `phase-spec.md`, after `wave-plan.md`, and after every wave report, before continuing. Opt-in — for when the human wants to inspect each step. |

Record the choice in `config.yml` as `mode:`. In `autobuild` a finishing
phase or wave never returns to the human for permission — the wave report and
the progress line are a notification the user can read later, not a prompt,
and the next wave has already begun. "Waiting" in `autobuild` only ever means
waiting on another agent (a Builder finishing, the Verifier passing), never
on the user.

---

## Input contract

### `phase-spec.md` — produced by Act 1, or hand-written

A numbered phase list. Each phase declares, verbatim in these fields:

- `goal` — one sentence
- `deliverables` — what code and tests exist when the phase is done
- `touches` — file globs the phase may modify (the fence for scope creep)
- `depends_on` — phase numbers whose deliverables this builds on, or `none`
- `tests` — the concrete checks that must pass for this phase
- `weight` — integer relative effort (1 trivial … 8 major build); drives the
  §Progress estimate and the default gate depth (§ACT 3)
- `gate` — optional override: `light` | `standard` | `full`. Omitted →
  derived from weight (1–2 light, 3–4 standard, 5+ full).
- `risk` — optional: `empirical` when the phase's approach rests on
  unverified behaviour of something the code review cannot predict — an
  optimizer's monotonicity, an external API's shape, a query planner, a real
  data distribution. Triggers a **spike** before the full build (§ACT 3
  step 1). The Interviewer tags these; when in doubt, tag it.

Anything the Interviewer could not get confirmed is tagged `ASSUMED —
correct me` inline, so the human fixes it in review, not in more Q&A.

### Phase 0 — contracts first

The Interviewer's first job after drafting phases is to **pull every shared
type, schema, interface, DB migration, and function signature that two or
more later phases need into a single Phase 0** (`weight` usually 1–2). Gate
it by what it actually lands: `gate: full` when Phase 0 carries real
behaviour (a migration that runs, a shared function with logic); `gate:
standard` when it is a behaviour-preserving extraction or a signature-only
stub — the test gate plus a fresh Reviewer already prove an extraction, and
a Verifier boot would exercise nothing new. Later phases then `depends_on: [0]`
instead of on each other, so the partition fans them out into one wave
instead of a chain. This is the highest-leverage speed move — do it
aggressively. A phase that still must depend on another phase's *behaviour*
(not just its interface) stays a real dependency.

### `.beehive/digest.md` — shared context, written once

The Interviewer, having explored the repo, writes a compact digest: the
conventions that matter, the file map for the area in play, the signatures
of helpers/types the phases will reuse, the test commands. Every later role
(Builder, Reviewer, Verifier) reads `digest.md` **instead of** re-deriving
that from raw files. One expensive exploration, many cheap reads. Keep it
under ~150 lines; it is a pointer sheet, not a copy of the codebase.

**Refresh:** after each wave merges, the Coordinator patches the digest with
what actually changed — new/renamed helpers, moved files, new signatures —
in a few lines, so the next wave's Builders read current reality, not the
intake-time snapshot. **This replaces, it doesn't accumulate:** fold the
change into the relevant existing section (conventions, file map,
signatures) instead of appending a new dated "what changed this wave" block
every time, and delete whatever a later wave has superseded. Every role
re-reads this file in full, every wave — a line you leave in gets paid for
again at every spawn. If a refresh finds the digest already past ~150 lines,
that refresh trims it back down, not just adds to it.

### `config.yml`

```yaml
spec: .beehive/phase-spec.md
conventions: "see CLAUDE.md / AGENTS.md"
mode: autobuild                        # autobuild (default) | review
hot_files:                            # 2 phases touching one → different waves
  - backend/app/main.py
context:                              # minimum every builder reads first — keep short
  - CLAUDE.md
test:                                 # full suite — every command exit 0 to pass a wave gate
  - pytest backend/                   # the Coordinator runs these commands concurrently
  - npm --prefix frontend test
test_quick: "pytest backend/ -k {phase_area}"  # optional; the fast subset a Builder runs while iterating
boot: "npm --prefix frontend run dev" # verifier only; optional
heavy_review: "/code-review ultra"    # human-run; the protocol never invokes it
models:                               # per-role model, routed by stakes (§Speed lever 9)
  strong: default                     # Coordinator, full-gate Reviewer, risk:empirical Builder
  cheap: <cheapest capable model>     # Verifier, standard-gate Reviewer, every routine
                                      # Builder — the Interviewer sets this; leaving it
                                      # `default` wastes real cost on every one of them
                                      # (a `light` gate has no separate Reviewer agent at
                                      # all — the Coordinator eyeballs it itself, so there
                                      # is no such thing as a "light-gate Reviewer" to route)
```

---

## Roles — defined by context isolation, not by tool

| Role | Receives | Must NOT have | Edits code |
|---|---|---|---|
| Interviewer | `brief.md`, `context`, repo read access | — | writes `phase-spec.md` / `config.yml` / `digest.md` only |
| Coordinator | everything | — | merges + docs; small glue/fix code if a gate still covers it (below) |
| Builder | its phase's section + `digest.md` + `conventions` | other phases' details | its phase's `touches` only |
| Reviewer | the wave diff + **this gate's phases** in `phase-spec.md` + `digest.md` | any Builder's session or reasoning; phases outside this gate | no |
| Verifier | the merged wave branch + **this gate's phases** in `phase-spec.md` + `digest.md` | any Builder's session or reasoning; phases outside this gate | no |

Roles read `.beehive/digest.md` for repo context instead of re-exploring
from raw files. A Builder still opens the specific files in its `touches`.

**Reviewer and Verifier MUST be a fresh agent instance / new session /
different person.** That isolation is the point — how this is done in
Claude Code is in `ADAPTERS.md`. Prompts for each role are in `ROLES.md`.

**A narrow exception for the Coordinator:** small, sequential glue or fix
work that doesn't need Builder isolation — a final polish phase, a fix
round after every Builder for the run is done — can be written by the
Coordinator directly instead of spawned as a fresh Builder, *when a
Review/Verify gate is still going to cover it* before the run reports
complete (the final combined gate, most often). This is not a way to skip
review — the code still has to land inside a wave whose gate checks it, same
as any Builder's diff; it differs only in who held the pen. A real run
measured this at zero marginal subagent cost for exactly this pattern (a
docs/polish phase plus two post-build fix rounds) — the saving is real, but
it only holds because a gate downstream still caught it.

---

## Procedure

### ACT 1 — Intake

1. Human writes `.beehive/brief.md`: everything they want, however messy.
   Their only writing obligation.
2. Interviewer runs (`ROLES.md` §Interviewer): explores the repo, drafts a
   phase breakdown, lists **its own** uncertainties and real forks, asks the
   human in batches of ≤6 (live if interactive; via `questions.md` if not).
3. Stops asking at **convergence** — every phase's `deliverables` /
   `touches` / `tests` writable with no unconfirmed assumption — or after 3
   rounds, tagging survivors `ASSUMED`.
4. Extracts the shared contracts into **Phase 0** (§Phase 0). Writes
   `phase-spec.md`, `config.yml`, and `digest.md`.
5. Asks the final question: **"Do you want to read the plan before I build,
   or should I just go ahead?"** → sets `mode`.
6. `mode: review` → human approves `phase-spec.md`. `mode: autobuild` →
   proceed straight into Act 2 in the same turn; do not stop to announce the
   spec or ask for a go-ahead.

### ACT 2 — Plan (Coordinator; deterministic)

```
parse phases; compute transitive depends_on closure
wave = 1; unassigned = all phases
while unassigned:
  ready = { p in unassigned : every depends_on is in an earlier wave }
  selected = []
  for p in ready, ascending phase number:
    if p.touches ∩ (∪ q.touches for q in selected) intersects a hot_file: skip p
    else selected += p
  assign selected → wave; unassigned -= selected; wave += 1
last wave = docs/polish only, alone
```

Phase 0 (contracts) is almost always wave 1 alone; a well-factored Phase 0
lets most of the rest land in wave 2 together. If the partition still comes
out as a long chain, that is a signal Phase 0 missed a shared contract or a
`hot_file` needs splitting (§Speed) — say so in `wave-plan.md`.

While computing the `depends_on` closure, also mark which phases are
**leaves** — nothing in any later wave `depends_on` them. A leaf is the only
kind of phase §Speed lever 4 allows gating later than its own wave; note each
wave's leaves in `wave-plan.md` even if you decide to gate them immediately
anyway, so the option is visible without recomputing the graph later.

Write `wave-plan.md`: the partition, and one line of reasoning per wave.
`mode: autobuild` → continue straight into Act 3 and spawn wave 1's Builders
in the same turn — the plan file is a record, not a checkpoint. `mode:
review` → stop for human sign-off first.

### ACT 3 — Build, for each wave N in order

Through every step below the Coordinator keeps `.beehive/wave-N/status` at
the current stage — one of `building | integrating | testing | review |
verify | blocked | merged` — so a resuming agent knows where the wave stopped.

1. **Build (concurrent, capped)** — spawn the wave's Builders up to the
   concurrency cap (default ~2–3 on a metered plan — §Speed lever 10), each
   on branch `wave-N/phase-K`, each in its own worktree/checkout
   (`ADAPTERS.md` for the mechanics — and for making the gitignored deps a
   build needs, like a venv or `node_modules`, present in each worktree). The
   Coordinator does not block on one before starting the next *within the
   cap* — launch up to the cap, then backfill one at a time as each finishes,
   rather than waiting for a full batch before starting more. A wave narrower
   than the cap is just spawn-all; the cap only matters once a wave is wider
   than it. A single-phase wave is just one Builder. Builder writes
   `.beehive/wave-N/build/phase-K.md` (≤15 lines:
   what changed, deviations, new files). While iterating, a Builder runs
   `config.test_quick` (fast subset); the full suite is the gate's job.
   **Spike first if `risk: empirical`.** Before writing the real
   implementation, the Builder (or the Coordinator in the main thread, if
   it's a few lines) writes the smallest throwaway script that exercises the
   unverified assumption against a real fixture, pastes the result into
   `.beehive/wave-N/spike-K.md` (≤10 lines), and only then proceeds. If the
   assumption is false, it stops and kicks the phase back to the human with
   the evidence — a re-spec now is a fraction of a full build→review→verify
   cycle discovering it at the end.
   **3-strike rule:** a Builder that can't get its own tests green after 3
   attempts stops and reports — the Coordinator reassigns or escalates,
   rather than letting it burn tokens flailing. **A Builder that dies
   mid-phase** (crash, rate limit, timeout) is *resumed* with its context
   intact, never cold-restarted — its partial work is on its branch, and a
   cold agent would re-explore and re-derive it. **Review/verify findings go
   back to the same Builder, context intact** (same-session continuation
   where the platform supports it — `ADAPTERS.md`), not to a fresh instance
   that re-reads the phase, the digest, and every touched file. Cold-start a
   fix Builder only if the original is unrecoverable. **A fix round costing
   nearly as much as the phase's original build is a signal the resume
   didn't actually stay warm** — check the per-role `tokens` field (§Report)
   rather than assume; a fix round should cost roughly what the findings
   take to read and act on, not a second full build.
   **A resume the Coordinator doesn't actively drive doesn't happen.**
   Dispatching a role and waiting is not a liveness check — a crashed agent
   that nobody re-pings stays crashed, silently, for the rest of the run.
   The Coordinator checks in on outstanding roles rather than trusting a
   notification that may never arrive (`ADAPTERS.md` for the mechanics); a
   role with no new output past a reasonable interval gets an explicit
   resume attempt before anything else proceeds. Record every crash→resume
   cycle in the wave report's `resume` field (§Report) as it happens, not
   from memory at gate time. A role that still hasn't come back when the
   gate is written is **stuck**, not silently dropped — the wave cannot
   report `status: passed` while any role is stuck.
2. **Integrate** — Coordinator merges phase branches → branch `wave-N-int`
   (not `wave-N` — that collides with the `wave-N/phase-K` refs).
   Resolves conflicts. A conflict that shows two phases disagreed on an
   interface is a Phase 0 gap: fix it, record it in the wave report.
3. **Test gate** — Coordinator runs the `config.test` commands (full suite),
   **concurrently** where they're independent (backend ∥ frontend). Raw
   output → `.beehive/wave-N/tests.log`. Any non-zero exit → hand the raw
   failure to the responsible Builder, repeat step 1 for that phase only.
   Never advance.
4. **Review + Verify — depth by gate.** Each phase's `gate` (explicit, or
   derived from `weight`) sets how hard this wave is checked. Take the
   **deepest** gate among the wave's phases:
   - `light` — Coordinator eyeballs the diff against `phase-spec.md`. No
     separate agent. No boot.
   - `standard` — fresh Reviewer on `git diff <working-branch>...wave-N-int`
     (`ROLES.md` §Reviewer) → `.beehive/wave-N/review.md`. No separate
     Verifier; the test gate stands in.
   - `full` — Reviewer **and** a fresh Verifier (`ROLES.md` §Verifier):
     clean checkout, `config.boot`, exercise the real user-facing paths this
     wave added, hand-check the numbers → `.beehive/wave-N/verify.md`. A
     clean checkout has no runtime state — the Verifier seeds what it needs
     from fixtures or a documented import, never the user's live data.
   **Deferring a leaf phase's gate.** If every phase in this wave is a leaf —
   nothing in a later wave `depends_on` it (§ACT 2 marks these in
   `wave-plan.md`) — this wave's Review+Verify can be skipped now and folded
   into a later wave's gate instead (the final wave, typically); the test
   gate (step 3) still applies before merge, same as any wave. When the
   fold-in wave runs its gate, the diff it reviews must span back to the last
   point the deferred wave was actually reviewed, not just its own wave — a
   plain `wave-N-int` diff against the now-current working branch would
   silently exclude the very phases you deferred. Note the deferral in
   `wave-plan.md` the moment you decide it. A wave with any non-leaf phase
   keeps its gate now — deferring a phase later work still builds on is the
   "review everything at the end" pattern §Speed rules out, not this. A
   deferred gate is a debt: the run can't report `100% · complete` while one
   is outstanding, and whichever wave clears it must cover every deferred
   wave's diff, not just its own.
   The Reviewer does **not** re-run `config.test` — it reads
   `.beehive/wave-N/tests.log` from step 3 (same tree, minutes old) and
   judges the diff. The Verifier re-runs `config.test` only if it has a
   concrete reason to distrust the log; its own job is the end-to-end
   exercise, which nothing else does. One full-suite tokenization per wave,
   not three.
   Any finding above trivial → Builder (same one, warm — step 1) → back to
   step 3. **The recheck gate is derived from the fix diff, not the phase.**
   A frontend / copy / single-constraint fix that doesn't touch verified
   substance → Coordinator eyeballs it (light). Only a fix that changes core
   logic re-triggers the phase's full gate.
   The liveness rule in step 1 applies here too — a fresh Reviewer or
   Verifier can die mid-check as easily as a Builder can, and gets the same
   active resume-and-track treatment, not a silent wait.
5. **Gate** — Coordinator writes `.beehive/wave-N/report.md`, starting with
   a parseable front-matter block, then the prose body:
   ```
   ---
   wave: N
   phases: [P0, P3]
   percent: 62
   status: passed        # passed | blocked
   blockers: []
   tokens: {builder: 41000, reviewer: 6000, verifier: 4000}  # n/a if the platform doesn't expose it
   mechanisms: {spiked: [], model_tiering: true, tests_reused: true, checkpointed: true, staggered: 3}
   resume: {}            # role: crash count, omit roles that never crashed; empty if none did
   stuck: []             # roles that crashed and never came back — must be empty for status: passed
   ---
   ```
   Body: diff stat, last ~20 lines of `tests.log`, review findings +
   resolutions, verify verdict (if run). Names `config.heavy_review` as the
   human's optional deep check. Patches `digest.md` with what changed
   (§digest). Updates `.beehive/progress.md` (§Progress). `tokens`,
   `mechanisms`, `resume`, and `stuck` are explained in §Report, below.
   `mode: autobuild` → merge `wave-N-int` to the working branch (it passed
   the test gate), then start wave N+1's **Build** immediately, in the same
   turn as writing the report, while this wave's Review+Verify run *in
   parallel* — they don't edit code, so a finding just feeds back as a fix
   on the already-merged phase. Do **not** post the report and wait for the
   user to tell you to continue — activating wave N+1 is automatic. Guard:
   if wave N has a `full` gate, N+1 may **build** in parallel but must not
   **merge** until N's Verify passes — the Coordinator waits on the Verifier
   agent for this, not on the user — so nothing compounds on an unverified
   foundation. Stop only on an unrecoverable block. `mode: review` → wait
   for approval.
   On a long run (4+ waves) the Coordinator's own context grows every wave —
   every dispatch prompt, every notification, every report it wrote. It may
   **drop that context between waves and resume from `.beehive/`**
   (§Resuming): `progress.md`, the newest report front-matter,
   `wave-N/status`. It holds no state the files don't. This caps the
   Coordinator's cost at roughly one wave's worth instead of the whole run's.

---

## Progress

After every wave's gate the Coordinator recomputes project completion and
appends one line to `.beehive/progress.md`. This is the "how much is done"
signal — it replaces stopping to ask the human.

Each phase in `phase-spec.md` carries a `weight` (integer, relative effort —
1 trivial, 8 a major build). Completion is weight-based, not phase-count,
because phases are uneven:

```
done      = Σ weight[p] for phases past their wave's gate (step 5)
building  = Σ weight[p] for phases merged but whose gate hasn't cleared  (× 0.6)
total     = Σ weight[p] for all phases
percent   = round( 100 * (done + 0.6 * building) / total )
```

`progress.md` line format:

```
wave 2 · 2026-08-30 14:07 · P1,P2,P3 done · P4 building · 47% · next: wave 3 (P4)
```

On the final wave, the last line reads `100% · complete`.

---

## Report

`percent` says how much is done. It says nothing about whether the run
actually behaved the way the protocol claims — that a `risk: empirical`
phase was really spiked, that the Verifier really ran on the cheap model,
that the Reviewer didn't quietly re-tokenize the whole suite. Two more
fields on the wave report (§ACT 3 step 5) close that gap, best-effort —
leave a field out or `n/a` rather than guess:

- **`tokens`** — per role, this wave's token usage, if the platform
  surfaces it (`ADAPTERS.md` has the mechanics). This is where the cost
  actually went, not an estimate from phase `weight`. A number sourced from
  a crash's resume call — where the resume reports only its own cost after
  the real pass had already finished before the crash notice arrived — is
  not the pass's true cost; flag it as unreliable in the report body rather
  than folding it into the total silently.
- **`mechanisms`** — what the Coordinator actually did this wave, not what
  the spec says it should do: which phases' `risk: empirical` spike ran
  (`spiked`), whether routing actually followed §Speed lever 9 — `strong`
  only for full-gate Reviewers and `risk: empirical` Builders, `cheap` for
  the Verifier, standard-gate Reviewers, and every routine Builder
  (`model_tiering`), whether the Reviewer read `tests.log`
  instead of re-running the suite (`tests_reused`), whether Reviewer/Verifier
  checkpointed their output instead of writing once at the end
  (`checkpointed`), and how many Builders ran at once (`staggered`). The
  Coordinator fills this from its own actions — it needs no extra
  inspection. A lever reading `false`/`0` across several waves is a signal
  the config or the Coordinator is skipping it, not that it doesn't apply.
- **`resume`** — per role that crashed this wave, how many crash→resume
  cycles it went through (§ACT 3 step 1). Roles that never crashed are
  omitted, not listed at `0`. This is the field that turns "a Builder that
  dies mid-phase is resumed" from an assumption into something the human
  can audit without reconstructing it from session logs after the fact.
- **`stuck`** — roles that crashed and, despite an active resume attempt
  (not just waiting), never came back. Non-empty `stuck` means the wave
  cannot report `status: passed` — a stuck Builder means unmerged work, a
  stuck Reviewer/Verifier means an ungated one, and moving on with either
  quietly is exactly the failure mode this field exists to stop. `blocked`
  with the stuck role named, same as any other unresolved gate.

On the final wave, after the last `progress.md` line, the Coordinator
writes `.beehive/summary.md`: `tokens` summed per role across every wave,
every wave's `mechanisms` merged into one run-wide checklist, and every
`resume`/`stuck` entry carried forward — so a run that needed several
crash recoveries to finish shows that plainly, instead of just `100% ·
complete`. One file the human reads once, instead of opening every wave
report.

---

## Speed

Wall-clock is bounded by the dependency graph, not the agent count (Amdahl):
if 80% of the work is a serial chain, infinite builders still only cut ~20%.
So beehive spends its effort making the graph *wider*, then runs the width
in parallel.

Levers, in the order they pay off — the first eight cut wall-clock, the last
two (and §Economy) cut token cost, which on a metered plan is the real
ceiling:

1. **Phase 0 contracts** (§Phase 0) — the single biggest one. Move every
   shared type/schema/signature into one small early phase so the rest
   `depends_on: [0]` and fan out into one wave instead of a chain.
2. **Merge phases that collide only on a `hot_file`.** If P3 and P4 both
   depend only on P0 and are serialized *purely* because they edit the same
   big file, hand both to **one** Builder as a combined phase — one pass
   over the file, no merge, faster than two builders + conflict resolution.
   The Interviewer should do this at spec time.
3. **Recommend splitting a `hot_file`** when three or more phases all edit
   it — a Phase 0 that carves the area into its own module unlocks
   parallelism for every later phase. Flag it; let the human decide.
4. **Risk-proportional gates** (§ACT 3 step 4) — a `weight: 1` docs phase
   does not need a fresh Reviewer and a Verifier boot. Save the full triad
   for the `weight: 5+` phases. And a fix-recheck is gated by the *fix*, not
   the phase — a copy tweak doesn't earn a second full Verifier pass. A
   **leaf phase** — nothing later `depends_on` it — can take this further
   than "lighter": deferred, folded entirely into a later wave's gate,
   because there is no downstream build left that can inherit a bug it
   hasn't caught yet (§ACT 3 step 4 has the mechanics and the guardrail). A
   phase anything later depends on keeps its own wave's gate regardless of
   weight — deferring *that* one is the "review everything at the end"
   pattern below, not this lever.
5. **Pipelined gates** (§ACT 3 step 5) — Review/Verify of wave N overlap
   the build of wave N+1.
6. **Tight inner loop** — `config.test_quick` while a Builder iterates,
   full suite only at the gate.
7. **`digest.md`** — one exploration shared by every role instead of each
   fresh agent re-reading the repo (also the main token lever, below).
8. **Concurrent test commands** — backend ∥ frontend at the gate, not
   back-to-back.
9. **Model tiering** (`config.models`) — route by **stakes, not role name**:
   `models.strong` for the Coordinator, full-gate Reviewers, and `risk:
   empirical` Builders — the three places a wrong call is expensive to
   discover late (the Coordinator's partition/merge/gate calls have nothing
   else behind them; a full-gate Reviewer is the only check on a foundation
   other phases build on; a bad spike green-lights the wrong approach for
   the whole phase); `models.cheap` for everything else — the Verifier
   (curl-and-paste evidence work), the standard-gate Reviewer (weight 3–4,
   lower stakes than full), and every *routine* Builder. Routine Builders
   default to `cheap`, not `strong`: once Phase 0 has landed the contracts,
   most phases are ordinary implementation, and the fresh Reviewer still
   catches what a cheaper model misses — that's the isolation guarantee
   doing its job, not a gap this leaves open. A `light` gate has no separate Reviewer agent at
   all — there is no third, cheaper tier to route; don't go looking for a
   "light-gate Reviewer" instance, it doesn't exist. This routing is not
   advisory. The Interviewer sets `models.cheap`; a run that leaves it
   `default` pays the strong-model rate for every curl-and-paste Verifier
   *and* every routine Builder. Doesn't cut token *count*, cuts cost.
10. **Stagger concurrency on a metered plan.** N concurrent Builders burn
    ~N× the token rate; a rate-limit mid-wave costs more than the
    parallelism saved, and a rate-limited Reviewer/Verifier is *pure* loss
    (no artifact until it finishes). Widen the graph at spec time, but run
    ~2–3 agents at once, not the whole wave width (§ACT 3 step 1 — spawn up
    to the cap, backfill the rest as slots free, never the whole wave at
    once). Reviewer and Verifier checkpoint their output as they go
    (§Economy) so a killed one resumes. A crash attributable to a spend/rate
    limit is a signal to cut the cap further for the rest of the run, not
    just to resume at the same width — a real run that lost 4 agents (3
    Builders and a Reviewer) to one simultaneous limit hit dropped to one
    agent at a time for everything after. Record the new cap as a dated note
    in `digest.md` (alongside `models.cheap`-style decisions) so later waves
    inherit it without re-deciding. **If crashes cluster at spawn or first
    resume rather than spread through a wave's runtime** (one run: 10 of 14),
    the trigger is likely the *simultaneity* of the spawn moment, not
    sustained concurrent width — stagger the dispatches themselves (a beat
    between each `Agent` call in a batch), not just a lower cap.

What does **not** help: adding builders to an existing chain (they idle),
or "build everything then review once at the end" (late-caught foundation
bugs force rework of everything above them — the per-wave gate is the cheap
place to catch them; a **leaf** phase is the one exception, lever 4, since it
has no "everything above it" left to protect). The de-risk equivalent:
committing a full Builder→Reviewer→Verifier cycle to a phase whose approach
was never checked against real data — spike it first (§ACT 3 step 1, `risk:
empirical`).

---

## Economy — token and code discipline (from ponytail)

Applies to every role. The point is a small correct diff and a short honest
report, not volume.

- **Understand first, then be lazy.** The reading is never shortened — trace
  every file the change touches and the real flow end to end. Only the
  *solution* gets minimized.
- **Reuse before writing.** A helper, type, or pattern already in the repo →
  use it. Re-implementing what lives a few files over is the most common
  waste. Stdlib and native platform features before any new code.
- **Shortest working diff, fewest files.** No abstraction with one caller, no
  scaffolding "for later", no config for a value that never changes. The
  smallest change *in the right place* — a guard in the shared function, not
  in every caller.
- **Minimal comments in the diff itself** — this is a rule ponytail doesn't
  carry, add it explicitly. No docstring that restates the signature, no
  line-by-line narration, no header banner. A comment earns its place only
  for a non-obvious constraint or a workaround a future reader would
  otherwise undo — that includes a `ponytail:` marker. Every line here is a
  line the Reviewer, the Verifier, and every later Builder reading the diff
  also pays to read; it's the same tax as unbounded prose, just paid inside
  the code instead of around it.
- **Context is rationed.** Each role gets only what its table row allows.
  Roles read `.beehive/digest.md` for repo context — one shared exploration,
  not one per agent. A Builder additionally opens the specific files in its
  `touches`; it does not take a repo tour. `config.context` stays short.
- **Reviewer sees the diff, not the files.** `git diff
  <working-branch>...wave-N-int`, plus `digest.md` and the spec — never the
  full source of every touched file.
- **Don't re-tokenize an unchanged test run.** The gate ran `config.test`
  and saved `tests.log`. The Reviewer reads that file; it does not re-run
  the suite on the same tree. Only the Verifier runs it again, and only if
  it distrusts the log. Same for the Coordinator across a fix loop — one
  full run per gate, re-run only what the fix touched.
- **Checkpoint long outputs.** Reviewer appends each finding to `review.md`
  as it lands; Verifier appends evidence as each check completes — not one
  write at the end. A rate-limited or crashed instance resumes from its own
  partial file instead of starting over.
- **Notes are bounded.** Builder note ≤15 lines. Reviewer findings are
  one-liners: `severity · file:line · problem · fix` — never restate the
  spec back. Verifier gives evidence, not narrative. Wave report is the
  fixed shape in §ACT 3 step 5, nothing added.
- **No unrequested prose.** If an explanation is longer than the code it
  defends, cut it. Explanation the human asked for (this protocol's reports,
  a walkthrough) is not debt — give that in full.
- **Bounded exploration.** The Interviewer reads at most ~15 files before
  drafting phases; past that it asks the human to point rather than
  spidering. A resuming agent reads report front-matter first and only
  opens the prose body it needs.
- **The build note is the handoff.** Reviewer reads the Builder's
  `phase-K.md` (≤15 lines) plus the diff — never a Builder's transcript.
- **Mark shortcuts, don't argue them.** A deliberate simplification gets a
  `ponytail:` comment naming the ceiling and the upgrade path, so Review
  doesn't re-litigate it: `# ponytail: single cash pool, per-account later`.
- **A deferral must land somewhere tracked** — a `ponytail:` comment or an
  item added to a later phase's spec. "Deferred to Phase 3" in a build note,
  with nothing in Phase 3, is an *untracked* deferral; the Reviewer treats
  it as a finding.
- **Never lazy about:** trust-boundary validation, error handling that
  prevents data loss, security, accessibility basics, atomicity the spec
  requires, or anything the spec explicitly asks for.
- **One runnable check per non-trivial unit**, matching the phase's `tests`
  field — no fixture sprawl or per-function suites unless the spec says so.

---

## Invariant rules

- Two phases sharing a `hot_file` never share a wave (unless merged into one
  phase for one Builder — §Speed lever 2).
- Never advance past a failing `config.test` command.
- Reviewer and Verifier never share context with a Builder.
- Builders receive only their phase's scope and touch only their `touches`.
- A Builder stops and reports after 3 failed attempts to green its tests —
  no indefinite flailing.
- Every role reads `digest.md`; nobody re-explores what it already covers.
- A `risk: empirical` phase spikes the unverified assumption before its full
  build; a false assumption goes back to the human, not into a full gate.
- Review/verify findings return to the Builder that wrote the phase, context
  intact — cold-start a fix Builder only if the original is unrecoverable.
- A crashed role is actively resumed and tracked (`resume` / `stuck` in the
  wave report), never silently left — a wave cannot report `status: passed`
  while any role is `stuck`.
- A deferred gate (§Speed lever 4) is recorded in `wave-plan.md`, not
  silent — the run cannot report `100% · complete` while one is still
  outstanding, and whatever wave clears it must cover every deferred wave's
  diff, not just its own.
- Coordinator-authored code (§Roles exception) is not exempt from a gate —
  it must fall inside a wave whose Review/Verify still covers it, same as
  any Builder's diff.
- Test results are pasted as raw output, never summarized as "passed".
- All cross-role handoff goes through `.beehive/` files.
- The final wave is docs/polish, alone.
- In `autobuild` the Coordinator never asks the user to confirm, approve, or
  "activate" a wave — not after the spec, not after the plan, not after a
  wave report, not between phases. Wave activation is automatic; the run
  halts only on an unrecoverable block.

---

## Resuming

The protocol assumes **one active run per `.beehive/`**. If `.beehive/` holds
a completed or abandoned prior run, archive it to `.beehive/archive/<feature>/`
before starting a new one.

No `phase-spec.md` yet → resume Act 1: if `.beehive/questions.md` holds the
human's answers, feed them to the Interviewer for its next round; otherwise
start intake fresh.

Otherwise read the front-matter of `.beehive/wave-*/report.md` newest-first
(cheap — `wave / phases / percent / status / blockers`) and the newest
wave's `status` file (`building | integrating | testing | review | verify |
blocked | merged`). Open a report's prose body only when you need the
detail. `progress.md` is the one-line history. The agent holds no state —
everything is on disk.

Because of this, a Coordinator on a long run can deliberately resume this
way between waves (§ACT 3 step 5) to keep its own context from growing with
every wave — not only after a crash.
