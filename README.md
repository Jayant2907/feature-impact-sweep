# feature-impact-sweep

An agent skill that checks whether a feature is correct **everywhere** it is read, computed, or shown — not only on the happy path you just built.

Compatible with **Claude Code** and **Cursor** (both use a `SKILL.md` skill folder).

## What this skill does

A feature is not done when its own path works. This skill hunts the bug classes a happy-path build misses:

1. **Path divergence** — the same quantity computed by several resolvers (run / preview / settle / export) that do not agree
2. **Denormalized / cached field drift** — a copy of the source of truth that feeds a computation, export, or decision
3. **Cross-screen / cross-session staleness** — state changed on screen A still stale on screen B
4. **Duplicates and orphans** — more than one live row per natural key; reverse ops that only flip a status
5. **Downstream / temporal impact** — consumers (exports, settlements, dashboards, period boundaries)
6. **Amount-to-amount verification** — one ground-truth record compared across preview, API, UI, and export
7. **UX / flow traps** — stranded controls, dead-ends, actions that will 409/400

It first builds an **impact map**, then verifies each edge. For a large surface, the seven checks can be fanned out as parallel sub-agents.

This is also the **"Impact" channel** used by `adversarial-review`.

## When to use it

Run it while or after building a feature that touches:

- a shared computation
- a denormalized or cached field
- a multi-step / multi-screen flow

Typical prompts: "cover surrounding modules", "check impact", "find workaround issues", "make sure nothing else broke".

## Install

### Cursor

**User-level:**

```bash
git clone https://github.com/Jayant2907/feature-impact-sweep.git ~/.cursor/skills/feature-impact-sweep
```

Windows (PowerShell):

```powershell
git clone https://github.com/Jayant2907/feature-impact-sweep.git "$env:USERPROFILE\.cursor\skills\feature-impact-sweep"
```

**Project-level:**

```bash
git clone https://github.com/Jayant2907/feature-impact-sweep.git .cursor/skills/feature-impact-sweep
```

### Claude Code

**User-level:**

```bash
git clone https://github.com/Jayant2907/feature-impact-sweep.git ~/.claude/skills/feature-impact-sweep
```

Windows (PowerShell):

```powershell
git clone https://github.com/Jayant2907/feature-impact-sweep.git "$env:USERPROFILE\.claude\skills\feature-impact-sweep"
```

**Project-level:**

```bash
git clone https://github.com/Jayant2907/feature-impact-sweep.git .claude/skills/feature-impact-sweep
```

Alternatively, copy the folder (it must contain `SKILL.md`) into the same locations.

Restart the agent session so the skill is picked up.

## How to run it

- `Run feature-impact-sweep on the feature I just built`
- `Check surrounding modules and impact for this change`

Adapt the "Notes to adapt per codebase" section in `SKILL.md` with your project's authoritative resolver, shared compute sink, and rebuild/restart commands. That is what makes the sweep precise on a new codebase.

## What's in this repo

```
feature-impact-sweep/
├── SKILL.md    # The 7-check sweep + output contract
└── README.md
```
