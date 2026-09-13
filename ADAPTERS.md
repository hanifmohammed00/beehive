# Beehive — Claude Code mechanics

Mechanics for running the protocol's roles inside Claude Code: **(a)** a
Builder scoped to one phase, **(b)** a Reviewer and Verifier that are fresh
instances with no exposure to the Builder, **(c)** parallel Builders when a
swarm is wide. Everything else is plain git + shell + files.

---

- **Coordinator**: the main session, or `/beehive` (skill wrapper reads
  `config.yml` and runs the Coordinator role).
- **Builder**: `Agent` tool, `subagent_type: general-purpose`, one call per
  phase. Parallel builders in one swarm → `isolation: "worktree"` +
  `run_in_background: true`, then collect. `model: {models.strong}` for a
  `risk: empirical` phase, `model: {models.cheap}` for every routine phase
  (§Speed lever 9 — resolve this per-phase, not one blanket model for every
  Builder in the swarm). On a metered plan, dispatch ~2–3 at a time rather
  than the whole swarm width — a rate-limit mid-swarm costs more than the
  parallelism saved, and a rate-limited Reviewer/Verifier (no artifact until
  it finishes) is a total loss paid twice.
  - **Before a batched dispatch, confirm distinctness — not after.** A real
    run lost ~200k tokens when two parallel Builder dispatches both
    inherited the Coordinator's own ambient cwd at the exact moment of a
    batched call — one agent caught the collision and correctly stopped,
    the other correctly refused a follow-up message claiming it was
    harmless, rather than trust an instruction over its own sandbox
    assignment. Before firing a batch of parallel `Agent` calls, list the
    worktree paths just created and check they're actually distinct;
    checking after dispatch is checking too late.
  - A `risk: empirical` phase: the Builder's first deliverable is the spike
    script + `.beehive/swarm-N/spike-K.md`, before the real build.
  - If `isolation: "worktree"` isn't available (it needs VCS hooks the
    environment may not have), fall back to **manual worktrees**: the
    Coordinator runs `git worktree add -b swarm-N/phase-K <dir> <base>` per
    phase and passes each Builder its `<dir>`. Symlink the gitignored build
    deps into each worktree first (`ln -s <repo>/backend/.venv <dir>/backend/.venv`,
    same for `node_modules`) or tests won't run. Background subagents are
    still fresh contexts, so isolation holds. Remove the worktrees
    (`git worktree remove --force`) and delete the merged branches after
    each swarm.
  - **Resuming a dead Builder, or feeding back fix findings**: `SendMessage`
    to the same subagent — its context and its partial work on the branch
    are intact. A fresh `Agent` call starts cold and re-explores the phase,
    the digest, and every touched file; only do that if the transcript is
    gone. Same for a rate-limited Reviewer/Verifier that checkpointed its
    partial `review.md` / `verify.md` — resume it, don't restart.
  - **Detecting a dead role, not just resuming one.** A background `Agent`
    call's completion notification is not guaranteed if the subagent's
    session dies outright — silence is not evidence it's still working.
    Don't wait indefinitely: if a swarm's `status` hasn't advanced and no new
    `build/phase-K.md` / `review.md` / `verify.md` line has appeared in
    longer than the phase would plausibly take, use `ListAgents` to check
    whether the subagent is still addressable. If it is, `SendMessage` to
    resume it; if it's gone, cold-start a fix instance per the rule above.
    Log each crash and its outcome into the swarm report's `resume` field
    (§Report) as it happens — a role that never comes back is `stuck` and
    blocks the gate. From one real run: of 7 role instances that crashed,
    the 5 running on an inherited model all resumed cleanly, while all 4
    running on an explicit `model:` override went permanently unresponsive.
    That run's explicit-model dispatches only existed because the human was
    manually correcting a model-tiering miss (below) — so the more likely
    story is reduced human attention on resumes during manual intervention,
    not the `model:` parameter itself. Still, until model tiering is
    actually resolved automatically and reliably, treat a manually-dispatched
    role as needing a liveness check sooner, not later.
- **Reviewer / Verifier**: a new `Agent` call — a fresh subagent starts cold,
  which satisfies isolation. Never reuse a builder subagent for review.
  Verifier and standard-gate Reviewer take `model: {models.cheap}` — pass
  it explicitly on the `Agent` call's `model` parameter; a `light` gate has
  no separate Reviewer agent, so there's no third case to handle. Point
  them at `swarm-N-int` (the integration branch) / its diff — or, if this
  gate is also covering a deferred earlier swarm (`PROTOCOL.md` §ACT 3 step
  4, §Speed lever 4), diff from the branch point *before* that deferred
  swarm instead of its immediate parent, so those phases are actually in
  scope and not silently excluded because they already merged.
- **Concurrent tests**: run the `config.test` commands as parallel
  background `Bash` calls, then read both results.
- **Heavy review**: the human runs `config.heavy_review` (e.g.
  `/code-review ultra`); the Coordinator only names it in the swarm report.
- **Swarm-to-swarm (autobuild)**: after writing a swarm report the Coordinator
  issues the next swarm's `Agent` calls in the same turn. It does not end its
  turn to let the user say "continue" — there is no confirmation step
  between swarms. The turn ends only at completion or an unrecoverable block.
- **Token/mechanism reporting (§Report)**: `mechanisms` needs no platform
  support — the Coordinator fills it from what it actually did (which model
  it dispatched each agent on, whether a spike ran, whether it told the
  Reviewer to skip re-running tests, how many Builders it ran at once), not
  from inspecting anything. `tokens` has **no tool-level source today** —
  neither the `Agent` tool's result nor `TaskOutput` returns a token count,
  only output text and status. Mark `tokens: n/a` per role rather than
  estimate. The one number a human can add by hand afterward is the
  session's own total cost/context display (e.g. `/cost`, or the terminal
  UI) — that's whole-session, not broken out per role, so it belongs in
  the human's own notes, not something the Coordinator can fill in.
