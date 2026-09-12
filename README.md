# Beehive

A platform-neutral protocol for building a multi-phase feature: interview a
messy brief into a phase spec, partition it into **waves** that build in
parallel where safe, and gate every wave behind an independent review and a
real end-to-end verification — autonomously, with a completion estimate
after each wave.

MIT-licensed. Vendor `PROTOCOL.md`, `ROLES.md`, `ADAPTERS.md` into any repo.

Works with Claude Code, Codex (CLI or Cloud), Cursor, Aider, or a human
coordinating several agent sessions. All state is plain files in the target
repo's `.beehive/` directory, so any agent on any platform can resume a run.

## Contents

| File | What it is |
|---|---|
| `SKILL.md` | Claude Code skill entry (`/beehive`) |
| `PROTOCOL.md` | The method — three acts, contracts-first Phase 0, wave partitioning, risk-proportional + pipelined gates, economy rules, progress formula, §Speed. Vendor this unchanged. |
| `ROLES.md` | Prompt templates for Interviewer, Coordinator, Builder, Reviewer, Verifier |
| `ADAPTERS.md` | How to instantiate the roles on each platform |
| `templates/` | Starting `config.yml`, `brief.md`, and `phase-spec.md` for a target repo's `.beehive/` |
| `examples/` | Worked examples |

## Install as a Claude Code skill

Global (available in every project):

```bash
ln -s "$(pwd)" ~/.claude/skills/beehive
```

Or per-project:

```bash
ln -s /path/to/beehive /path/to/your-repo/.claude/skills/beehive
```

Then `/beehive` in that project. A copy works too — re-copy on update.

## Use without a skill (Codex etc.)

Point the repo's `AGENTS.md` at the protocol:

```
Multi-phase features follow beehive: PROTOCOL.md — interview to a phase
spec, build in waves with independent review + verify gates.
```

Vendor `PROTOCOL.md`, `ROLES.md`, `ADAPTERS.md` into the repo (e.g. under
`.beehive/`) so the agent can read them.

## Bootstrapping a run

1. `mkdir .beehive` in the target repo; copy `templates/config.yml` to
   `.beehive/config.yml` and fill it (hot files, test commands, context).
2. Copy `templates/brief.md` to `.beehive/brief.md` and fill it in —
   everything you want, however messy.
3. Run `/beehive intake` (or hand `ROLES.md` §Interviewer to any agent, or
   write `.beehive/phase-spec.md` yourself from `templates/phase-spec.md`).
   It interviews you, then writes `.beehive/phase-spec.md` (with a
   contracts-first Phase 0 so the rest fans out in parallel),
   `.beehive/digest.md` (shared context every later agent reads instead of
   re-exploring), and asks whether to just build or let you review first.
4. It partitions into waves and builds — Builders in a wave run
   concurrently, gates are as deep as each phase's risk warrants, and the
   next wave's build overlaps the current wave's review. A phase whose
   approach rests on unverified real-world behaviour is spiked against real
   data before the full build, so a wrong design is caught at spec time, not
   after a full review-and-verify cycle. Reports after each wave — including
   token usage per role, which economy levers actually fired, and any
   crash→resume cycles a role went through, not just what the spec says
   should happen — and writes a completion percent to `.beehive/progress.md`.
   A role that crashes gets actively resumed, not silently left; one that
   never comes back blocks the wave instead of vanishing from the count. In
   the default `autobuild` mode it never stops to ask you to approve or
   start a wave — it just keeps going, halting only on an unrecoverable
   block. On the final wave it writes `.beehive/summary.md`: the whole run's
   token usage, lever checklist, and any crash history in one file.
