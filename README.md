# Beehive

A Claude Code skill for building a multi-phase feature: interview a messy
brief into a phase spec, partition it into **swarms** that build in parallel
where safe, and gate every swarm behind an independent review and a real
end-to-end verification — autonomously, with a completion estimate after
each swarm.

MIT-licensed. All run state is plain files in the target repo's `.beehive/`
directory, so a run can be resumed across sessions.

## Contents

| File | What it is |
|---|---|
| `SKILL.md` | Claude Code skill entry (`/beehive`) |
| `PROTOCOL.md` | The method — three acts, contracts-first Phase 0, swarm partitioning, risk-proportional + pipelined gates, economy rules, progress formula, §Speed. |
| `ROLES.md` | Prompt templates for Interviewer, Coordinator, Builder, Reviewer, Verifier |
| `ADAPTERS.md` | How to spawn Builders/Reviewers/Verifiers as Claude Code subagents |
| `templates/` | Starting `config.yml`, `brief.md`, and `phase-spec.md` for a target repo's `.beehive/` |
| `examples/` | Worked examples |

## Install

Global (available in every project):

```bash
ln -s "$(pwd)" ~/.claude/skills/beehive
```

Or per-project:

```bash
ln -s /path/to/beehive /path/to/your-repo/.claude/skills/beehive
```

Then `/beehive` in that project. A copy works too — re-copy on update.

## Bootstrapping a run

1. `mkdir .beehive` in the target repo; copy `templates/config.yml` to
   `.beehive/config.yml` and fill it (hot files, test commands, context).
2. Copy `templates/brief.md` to `.beehive/brief.md` and fill it in —
   everything you want, however messy.
3. Run `/beehive intake` (or paste `ROLES.md` §Interviewer into a Claude
   Code chat directly, or write `.beehive/phase-spec.md` yourself from
   `templates/phase-spec.md`).
   It interviews you, then writes `.beehive/phase-spec.md` (with a
   contracts-first Phase 0 so the rest fans out in parallel),
   `.beehive/digest.md` (shared context every later agent reads instead of
   re-exploring), and asks whether to just build or let you review first.
4. It partitions into swarms and builds — Builders in a swarm run
   concurrently (up to a cap, so a wide swarm doesn't crash them all onto a
   rate limit at once), and gates are as deep as each phase's risk warrants —
   a phase nothing later depends on can even have its gate deferred and
   folded into a later one — while the next swarm's build overlaps the
   current swarm's review. A phase whose approach rests on unverified
   real-world behaviour is spiked against real data before the full build,
   so a wrong design is caught at spec time, not after a full
   review-and-verify cycle. Reports after each swarm — including
   token usage per role, which economy levers actually fired, and any
   crash→resume cycles a role went through, not just what the spec says
   should happen — and writes a completion percent to `.beehive/progress.md`.
   A role that crashes gets actively resumed, not silently left; one that
   never comes back blocks the swarm instead of vanishing from the count. In
   the default `autobuild` mode it never stops to ask you to approve or
   start a swarm — it just keeps going, halting only on an unrecoverable
   block. On the final swarm it writes `.beehive/summary.md`: the whole run's
   token usage, lever checklist, and any crash history in one file.
