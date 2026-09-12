# Beehive — platform adapters

The protocol needs three things a platform must provide its own way:
**(a)** a Builder scoped to one phase, **(b)** a Reviewer and Verifier that
are fresh instances with no exposure to the Builder, **(c)** optionally,
parallel Builders. Everything else is plain git + shell + files.

---

## Claude Code

- **Coordinator**: the main session, or `/beehive` (skill wrapper reads
  `config.yml` and runs the Coordinator role).
- **Builder**: `Agent` tool, `subagent_type: general-purpose`, one call per
  phase. Parallel builders in one wave → `isolation: "worktree"` +
  `run_in_background: true`, then collect. `model: {models.strong}`. On a
  metered plan, dispatch ~2–3 at a time rather than the whole wave width — a
  rate-limit mid-wave costs more than the parallelism saved, and a
  rate-limited Reviewer/Verifier (no artifact until it finishes) is a total
  loss paid twice.
  - A `risk: empirical` phase: the Builder's first deliverable is the spike
    script + `.beehive/wave-N/spike-K.md`, before the real build.
  - If `isolation: "worktree"` isn't available (it needs VCS hooks the
    environment may not have), fall back to **manual worktrees**: the
    Coordinator runs `git worktree add -b wave-N/phase-K <dir> <base>` per
    phase and passes each Builder its `<dir>`. Symlink the gitignored build
    deps into each worktree first (`ln -s <repo>/backend/.venv <dir>/backend/.venv`,
    same for `node_modules`) or tests won't run. Background subagents are
    still fresh contexts, so isolation holds. Remove the worktrees
    (`git worktree remove --force`) and delete the merged branches after
    each wave.
  - **Resuming a dead Builder, or feeding back fix findings**: `SendMessage`
    to the same subagent — its context and its partial work on the branch
    are intact. A fresh `Agent` call starts cold and re-explores the phase,
    the digest, and every touched file; only do that if the transcript is
    gone. Same for a rate-limited Reviewer/Verifier that checkpointed its
    partial `review.md` / `verify.md` — resume it, don't restart.
  - **Detecting a dead role, not just resuming one.** A background `Agent`
    call's completion notification is not guaranteed if the subagent's
    session dies outright — silence is not evidence it's still working.
    Don't wait indefinitely: if a wave's `status` hasn't advanced and no new
    `build/phase-K.md` / `review.md` / `verify.md` line has appeared in
    longer than the phase would plausibly take, use `ListAgents` to check
    whether the subagent is still addressable. If it is, `SendMessage` to
    resume it; if it's gone, cold-start a fix instance per the rule above.
    Log each crash and its outcome into the wave report's `resume` field
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
  them at `wave-N-int` (the integration branch) / its diff.
- **Concurrent tests**: run the `config.test` commands as parallel
  background `Bash` calls, then read both results.
- **Heavy review**: the human runs `config.heavy_review` (e.g.
  `/code-review ultra`); the Coordinator only names it in the wave report.
- **Wave-to-wave (autobuild)**: after writing a wave report the Coordinator
  issues the next wave's `Agent` calls in the same turn. It does not end its
  turn to let the user say "continue" — there is no confirmation step
  between waves. The turn ends only at completion or an unrecoverable block.
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

## Codex — CLI (`codex` / `codex exec`)

- **Coordinator**: the interactive `codex` session.
- **Builder**: `codex exec` per phase, sequential, prompt = `ROLES.md`
  §Builder with slots filled. Each writes its own branch.
- **Reviewer / Verifier**: a *separate* `codex exec` run pointed at the
  diff (`git diff <working-branch>...wave-N-int`) — new process, no shared
  context, so isolation holds. Prompt from `ROLES.md`.
- **Parallel**: not native; the human can launch multiple Codex Cloud tasks
  (below) for one wave's phases and merge the branches.

## Codex — Cloud tasks

- **Builder**: one cloud task per phase in the wave, prompt from `ROLES.md`
  §Builder — these run in parallel natively, each producing a branch/PR.
- **Integrate**: the human (or Coordinator task) merges the phase branches.
- **Reviewer / Verifier**: a separate cloud task per role, scoped to the
  merged branch's diff.
- No skill needed — point `AGENTS.md` at `.beehive/PROTOCOL.md`.

## Cursor / Aider / other single-session tools

- **Coordinator + Builder**: the session, one phase at a time.
- **Reviewer / Verifier**: a new chat (Cursor) or a fresh `aider` invocation
  (`aider --message "$(fill ROLES.md §Reviewer)"`) against the diff — new
  context = isolation.
- **Parallel**: none; run phases sequentially within a wave.

## Manual multi-agent (any mix of vendors)

- Human is the Coordinator. Open one agent window per phase for Builders,
  merge their branches, then open a *different* window (any vendor) for the
  Reviewer and another for the Verifier, pasting the role prompts.
- The `.beehive/` files are the shared state — every window reads and
  writes there.

---

## AGENTS.md / CLAUDE.md pointer

Add once, so any tool entering the repo finds the method:

```
Multi-phase features follow .beehive/PROTOCOL.md — interview to a
phase-spec, build in waves with independent review + verify gates.
```
