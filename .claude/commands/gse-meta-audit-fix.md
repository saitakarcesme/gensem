---
description: "Post-audit fix session for the GSE-One methodology repository. Consumes the canonical registry _LOCAL/audit/audit.json produced by /gse-meta-audit: verifies findings against LLM fabrication (per-cluster verification pass), persists verdicts in place, applies CONFIRMED fixes cluster by cluster, and closes with the full build pipeline. Operationalizes CLAUDE.md > Post-audit fix workflow."
argument-hint: "[--clusters <id,id,...> | --all-open]"
---

# /gse-meta-audit-fix — Post-audit fix session (verification → application → release)

Arguments: $ARGUMENTS

## Scope

This command drives the **fix session that follows a `/gse-meta-audit` run**. It is the operational form of the protocol documented in `CLAUDE.md > Post-audit fix workflow` (canonical rationale — read it if in doubt). It operates exclusively on the single canonical registry `_LOCAL/audit/audit.json` (see "Audit output contract" in CLAUDE.md): verdicts, statuses, and resolutions are updated **in place** — never in side files.

Golden rule: **never apply a fix that has not passed the verification pass.** The 2026-04-22 session measured 33% false positives (4 of 12 clusters) among unverified LLM findings.

## Required readings

1. `.claude/agents/methodology-auditor.md` — adopt this role for every verification sub-agent (principles 1, 8, 9, 10, 11)
2. `CLAUDE.md > Post-audit fix workflow` — canonical protocol + version bump matrix
3. `_LOCAL/audit/audit.json` — the registry being processed

## Workflow

### Phase 0 — Context detection

- `_LOCAL/audit/audit.json` must exist and contain findings with `status: open`. If absent, abort: run `/gse-meta-audit` first.
- Render a quick view for the maintainer if useful: `python3 gse-one/audit.py --render` (throwaway `/tmp` markdown — never persisted in-repo).

### Phase 1 — Cluster selection

Determine which clusters to address this session:
- `--clusters <id,...>`: only the named clusters
- `--all-open` (or no args): every cluster containing at least one `status: open` finding of severity error or warning

The **cluster is the unit of verification and of application** — not the individual finding.

### Phase 2 — Parallel anti-false-positive verification

For each selected cluster, spawn ONE methodology-auditor sub-agent — ALL in a SINGLE message with multiple Agent tool invocations (clusters are independent). Verification prompt per cluster:

```
You are the methodology-auditor (defined in .claude/agents/methodology-auditor.md — principles 1, 8, 9, 10, 11 apply: evidence, verify-before-report, anti-rigidity, structured verdict, product-vs-source dualism).

MISSION: verify cluster <cluster-id> findings from the registry _LOCAL/audit/audit.json.
For each finding, produce a structured verdict per Principle 10.

FINDINGS TO VERIFY:
- <finding 1 with id + location + claim + proposed fix>
- <finding 2 ...>

For each finding, output (YAML):
  finding_id: <AUD-... id>
  verdict: CONFIRMED | FALSE_POSITIVE | NEEDS_REFINEMENT | SCOPE_CHANGE
  verdict_rationale: "<short explanation>"
  current_state: "<evidence as file:line excerpts>"
  proposed_fix: "<confirmed, adjusted, or 'no action — false positive'>"
  anti_rigidity_check: "<is the divergence deliberate? is the fix forcing uniformity?>"

Return YAML list, no preamble. Cite file:line for every claim.
```

### Phase 3 — Persist verdicts into the registry

Write the collected verdicts to a JSON file (e.g. `/tmp/gse-verdicts.json`: a list of `{finding_id | finding_title, verdict, verdict_rationale, current_state, proposed_fix}`) and apply them in place:

```
python3 gse-one/audit.py --set-verdicts /tmp/gse-verdicts.json
```

This updates each matched finding's `verdict`/`status` in `_LOCAL/audit/audit.json` (FALSE_POSITIVE → `wontfix` + a `detector_issues[]` entry; SCOPE_CHANGE → `deferred`), sets `meta.verify_run_at`, and recomputes the summary. **Do NOT write a `verify-*.md` file.**

### Phase 4 — Consolidation + bulk user validation

Present ONE structured message with four clearly separated groups:

1. **CONFIRMED** — actionable batch with exact file:line changes
2. **FALSE_POSITIVE** — each with root cause; if the detector can be fixed at source (`audit.py` regex, `.claude/audit-jobs.json` check wording), add a work item
3. **NEEDS_REFINEMENT** — adjusted fix proposal alongside the original
4. **SCOPE_CHANGE** — reclassification and new scope (defer to release X, bundle with contract change Y)

Close with a single-default question (Communication style Rule 2): "ok to apply all CONFIRMED + NEEDS_REFINEMENT (as adjusted), skip FALSE_POSITIVE, defer SCOPE_CHANGE". Do NOT ask per-finding — the verification pass is what earns the bulk-validation trust.

### Phase 5 — Cluster-based application

Apply fixes **grouped by cluster theme** (same root cause), not by file. A cluster may touch multiple files atomically. Update each processed finding's `status`/`resolution` in the registry as you go. Isolate contract changes (signatures, return shapes, state schemas) into their own dedicated release — never bundle with unrelated fixes (v0.55.0 pattern).

### Phase 5.5 — Backlog closure

Recommendations that survive the session as future work are promoted to the durable backlog (`python3 gse-one/audit.py --backlog-promote`), and items whose chantier was actually done in this session are closed (`--backlog-set BKL-ID=done`). The backlog (`_LOCAL/audit/backlog.json`) is the cross-run registry — see CLAUDE.md > Audit output contract.

### Phase 6 — Release

Run the full build pipeline (see `/gse-release` and CLAUDE.md > Build pipeline): version bump per the matrix in CLAUDE.md > Post-audit fix workflow Phase 4, CHANGELOG entry, generate `--verify`, commit, push, tag. The CHANGELOG of the resolving release MUST document each FALSE_POSITIVE: what was claimed, what was true, detector root cause, whether the detector was fixed at source in the same release.
