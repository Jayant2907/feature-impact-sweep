---
name: feature-impact-sweep
description: >-
  Run this WHILE or AFTER building/changing a feature that touches a shared
  computation, a denormalized/cached field, or a multi-step / multi-screen flow.
  It is a surrounding-module + cross-cutting IMPACT sweep: it hunts the bug
  classes that a happy-path feature build misses — the same computation
  resolved differently across code paths, denormalized fields that drift from
  the authoritative source, state changed in one screen not reflected in
  another, duplicate/orphan records, downstream/temporal impact, and
  preview-vs-final-record mismatches. Invoke when the user says "cover
  surrounding modules", "check impact", "find workaround issues", "make sure
  nothing else broke", or after any feature that adds a new way to compute an
  existing quantity. Also runs as the "Impact" channel of a code-review or
  PR-review workflow.
---

# Feature Impact Sweep

A feature is not done when the happy path works. It is done when every OTHER
path that touches the same data agrees with it. This skill is the checklist that
turns "I built X" into "X is correct everywhere X is read, computed, or shown."

Canonical origin (keep as the worked example): building a **period-shift**
feature in a scheduling app. The feature itself computed correctly on the
primary RUN path, yet the *surrounding* surfaces were wrong — the Preview
screen showed calendar days (31) while the finalized record showed working
days (26); a downstream settlement path silently used a wrong proration
divisor; a reopened record still showed "APPROVED" on another screen. None of
these were in the feature's own code. That gap is what this sweep closes.

## Where this runs

Invoke it directly for a feature you just built, **and** wire it in as the
**"Impact" channel** of a broader adversarial/code review — dispatched as a
`general-purpose` sub-agent loading this skill whenever the reviewed change
builds or modifies a feature, returning findings in the review's standard JSON
schema with `"channel": "impact"`.

## How to run it

Work through the seven checks below for the feature you just built or changed.
For a large surface, the checks are independent — fan them out as parallel
sub-agents (one per check) and reconcile; auditing "every reader of field X" is
embarrassingly parallel. Produce an **impact map** first (what does this feature
read / write / display, and where else is that same thing read/written/shown?),
then verify each edge of that map.

### 1. Path divergence — one quantity, many resolvers
The #1 bug class. The same output (a divisor, a rate, an eligibility flag) is
often computed in several places: the primary RUN path, a PREVIEW/SIMULATE
path, a FINAL-SETTLEMENT path, a RECOMPUTE path, an EXPORT/report path.
- List every path that produces the feature's quantity. `grep` the resolver
  function name and the field it returns.
- Confirm they share ONE resolver, or prove they agree. Common trap: the
  primary path threads the authoritative input (e.g. `getActiveRecord(asOf)`),
  but simulate/preview/settlement paths omit it and fall back to a default or
  a stale field.
- Fix at the **shared sink** (the lowest common function), not per-caller, so
  future callers inherit the correct behaviour. Guard so callers that already
  pass the input are untouched.
- *Example:* `buildContext`/`buildSimulationContext` both funnel into
  `buildInternal`; the fix belonged in `buildInternal`, not each caller.

### 2. Denormalized / cached field drift
Any field that duplicates an authoritative source WILL drift.
- Identify the authoritative source (an effective-dated table read via a
  resolver) vs the denormalized copy (a column on the parent entity).
- `grep` EVERY reader of the denormalized field. Classify each: SAFE (uses the
  resolver / is a write / is display-only with no downstream impact) vs RISKY
  (feeds a computation, export, or decision off the stale copy). Rank RISKY by
  blast radius (money/critical-value > metadata > display).
- Check whether the feature's own write path updates BOTH sources.
- *Example:* `record.assignedPlanId` drifted from the active assignment; the
  settlement path read the stale copy for the proration divisor (a numeric
  bug), while the primary run read the resolver.

### 3. Cross-screen / cross-session state staleness
State mutated on screen A must be reflected on screen B.
- For every screen showing the feature's state: does it refetch when re-entered
  / re-focused, or cache / auto-restore stale state?
- Watch for "auto-restore" logic that loads a prior record and never clears it
  when the record changes elsewhere; add refetch-on-focus and clear-on-stale.
- Watch for a status shown from one source on A and a different source on B.
- *Example:* reopening a record on an admin/output screen left the primary
  workflow screen showing the old APPROVED status and blocking a rerun.

### 4. Data integrity — duplicates & orphans
- Query for duplicates of the feature's records (>1 live row per natural key,
  e.g. entity × cycle × period). Duplicate/inconsistent rows make ALL
  downstream verification unreliable — clean them before trusting numbers.
- Check that reverse operations (reopen, cancel, delete) actually reverse their
  side effects (running totals, generated documents, ledger entries,
  paid/settled flags), not just flip a status.

### 5. Downstream / temporal impact
- Enumerate modules that CONSUME the feature's output: any compliance or
  regulatory surface, accruals, final settlement, exports/registers,
  dashboards.
- For period-shifted or effective-dated features: calendar-boundary splits,
  fiscal-year boundaries, mid-period joiners/leavers, period-attribution in
  each consumer.

### 6. Amount-to-amount verification across surfaces
"It renders" is not verification. Establish ONE ground truth and diff everything
against it.
- Ground truth = a real finalized record produced by the correct engine path.
- Confirm the SAME numbers appear in: preview, simulate/what-if, the raw
  API/context endpoint, the FE display, and the export/register.
- Any mismatch is path-divergence (check 1) or a stale read (check 2) — never
  hand-wave it as rounding.

### 7. UX / flow traps
- Disabled controls that strand the user (a locked selector with no escape).
- Multi-step dead-ends (having to press Back twice; no explicit "redo from
  start").
- Actions offered that will 409/400 downstream (offer the correct action for the
  current state instead).

## Output

Deliver: (a) the impact map, (b) a ranked findings table
`[file:line | what it does | SAFE/RISKY | blast radius | fix]`, (c) the fixes
you were confident on (fix at the shared sink; guard; re-run relevant tests +
rebuild/restart before claiming verified), and (d) residual follow-ups you did
not fix, flagged with why. When run as a review channel, return the standard
review JSON with `"channel": "impact"`.

## Notes to adapt per codebase

This checklist generalizes well, but its power comes from a few concrete
invariants specific to your own system. Before running it seriously on a new
codebase, spend a few minutes filling in your own version of these — then keep
them here as a living appendix:

- **Authoritative resolver vs. denormalized copy.** Name the one function that
  is the source of truth for your #1 shared quantity, and the field(s) known
  to drift from it.
- **Shared compute sink.** Name the lowest-level function every "produce this
  quantity" path should funnel through, and which callers are known to
  self-resolve instead (a common source of divergence).
- **Where money/critical-value truth actually lives.** Usually a small set of
  tables/entities, not the preview or simulation output — name them.
- **Rebuild/restart discipline.** After a backend change, what's the exact
  command sequence (build → restart → re-hit the real endpoint) before a fix
  can be claimed verified? Stale build artifacts are the first suspect for "the
  fix didn't work."
- **Emission/outbox invariants.** If your system projects facts into a
  reporting or event-outbox table, name the one invariant ("every mutating
  path on entity X must emit an event") and the full list of mutating paths to
  cross-check for a missing emit.
- **Reversal symmetry across duplicate paths.** When one destructive operation
  has two implementations (e.g. a bulk path and a per-item path), diff the set
  of side effects each reverses. Asymmetry is a real bug on the weaker path —
  treat any comment/docstring claiming "reverses X" as a claim to verify
  against the code, not a fact.
- **Guard-bypass via a shared primitive.** A guard on a wrapper method is
  bypassable if the underlying primitive it delegates to is ALSO exposed as
  its own endpoint with no guard. Always check the primitive + its direct
  route, not just the guarded wrapper.
- **Visibility for "special-cased" records.** A record that skips the normal
  compute path (manual entry, bulk import, materialized override) must still
  write everything downstream readers expect (status flags, snapshot rows,
  ledger entries) or it silently vanishes from reports/registers built off
  those fields instead of the record itself.
- **Sample-authoritativeness for generated file formats.** "Byte-exact to the
  sample" is only as good as the sample. Confirm the committed oracle file IS
  the real authoritative reference (and its format matches what the consumer
  actually expects), not a repo stand-in.
