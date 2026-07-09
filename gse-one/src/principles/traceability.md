# P6 — Traceability

**Category:** Foundations
**Principle:** Every artefact is traceable to its origin and related artefacts through a unique identifier and explicit trace links.

## Description

Traceability is the ability to follow an artefact from its origin (why it exists) through its implementation (how it was built) to its verification (how we know it works). In GSE-One, traceability is achieved through a systematic identification scheme and explicit trace links recorded in YAML frontmatter.

Every artefact receives a unique identifier upon creation. These identifiers are never recycled: if artefact REQ-012 is deprecated, no future artefact will ever be assigned REQ-012. Trace links connect artefacts across types, enabling impact analysis ("if this requirement changes, which tests are affected?") and coverage analysis ("which requirements have no tests?").

## Operational Rules

1. **ID allocation prefixes** — Each artefact type has a designated prefix (12 prefixes, canonical per spec §P6 — Traceability):

   | Prefix | Artefact Type | Example |
   |--------|---------------|---------|
   | `REQ-` | Requirement | REQ-001, REQ-042 |
   | `DES-` | Design decision | DES-001, DES-015 |
   | `TST-` | Test definition | TST-001, TST-030 |
   | `TCP-` | Test campaign report (execution event) | TCP-007 |
   | `RVW-` | Review finding | RVW-001, RVW-008 |
   | `DEC-` | Decision | DEC-001, DEC-003 |
   | `TASK-` | Work item (backlog + sprint tasks — unified) | TASK-001, TASK-055 |
   | `INT-` | Project intent | INT-001 |
   | `OQ-` | Open question (pending resolution) | OQ-003 |
   | `SRC-` | External source | SRC-001, SRC-003 |
   | `LRN-` | Learning note | LRN-001, LRN-012 |
   | `AUD-` | Audit finding (methodology drift — *meta-scope, produced by `/gse:audit`, not a project artefact*) | AUD-003 |

2. **ID uniqueness** — IDs are unique within the project scope. The agent maintains a counter per prefix. IDs are NEVER recycled. If REQ-005 is deprecated, the next requirement is REQ-006 (or whatever the next unused number is), never REQ-005 again. There is no separate ID registry file: existing IDs are enumerated from `.gse/backlog.yaml` and artefact frontmatter, and surfaced by the dashboard.

3. **Trace link types** — Seven typed link categories are used in frontmatter (canonical per spec §P6 — Traceability):

   | Link type | Meaning | Example |
   |-----------|---------|---------|
   | `derives_from` | "I come from..." | REQ derives from TASK, DES derives from REQ |
   | `implements` | "I realize..." | Code implements DES; a sprint tests doc covers [REQ, DES] (doc-level rollup) |
   | `validates` | "I validate..." | TST validates a REQ acceptance criterion (validation tests) |
   | `enforces` | "I enforce..." | Policy TST enforces a DEC or a design structural rule |
   | `tested_by` | "I am verified by..." | REQ/DES points back at its TST (inverse link, see spec §12.2 — Artefact Metadata) |
   | `decided_by` | "This choice is justified by..." | DES justified by DEC |
   | `related_to` | "Related to..." | Catch-all for weak/informational links |

   ```yaml
   traces:
     derives_from: []    # This artefact was derived from these artefacts
     implements: []      # This artefact realizes these artefacts
     validates: []       # This test validates these acceptance criteria
     enforces: []        # This policy test enforces these decisions/rules
     tested_by: []       # This artefact is verified by these tests (inverse link)
     decided_by: []      # This artefact was shaped by these decisions
     related_to: []      # Informational relationship (no formal dependency)
   ```

   Only include the link types that apply — omit empty ones for brevity:
   ```yaml
   # Minimal example — only 1 link type needed:
   traces:
     derives_from: [REQ-007]
   ```

4. **Bidirectional consistency** — Trace links imply a reverse relationship. If artefact A says `implements: [REQ-007]`, then REQ-007 is implemented by A. The agent MUST maintain bidirectional consistency when creating or updating trace links.

   | If A says... | Then B should reference A via... |
   |-------------|--------------------------------|
   | `derives_from: [B]` | B is derived into A |
   | `implements: [B]` | B is implemented by A |
   | `validates: [B]` | B is validated by A (typically recorded as `tested_by: [A]`) |
   | `enforces: [B]` | B is enforced by A |
   | `tested_by: [B]` | B verifies A (B typically records `validates: [A]`) |
   | `decided_by: [B]` | B justifies A |
   | `related_to: [B]` | B is related to A |

5. **artefact_type field** — The type vocabulary has two scopes (do not mix them):
   - **TASK `artefact_type`** (what a work item produces, per spec §P6 — Traceability): `code` | `requirement` | `design` | `test` | `doc` | `config` | `import` | `spike` (8 values; `spike` is complexity-boxed, max 3 points, non-deliverable).
   - **Document-level `artefact_type`** (flat frontmatter of standalone one-artefact files, per spec §12.2 — Artefact Metadata): `intent` (INT-NNN) | `learning` (LRN-NNN) | `external-source` (SRC-NNN). Section-level documents (requirements, design, tests, review…) use the nested `gse: type:` form instead — see spec §12.2 for the two storage patterns.

6. **Impact analysis** — Before modifying any artefact, the agent MUST check its trace links to identify potentially impacted artefacts. If impacts are found, they are reported to the human:
   ```
   Agent: Modifying REQ-012 will impact:
   - DES-007 (implements REQ-012)
   - TST-030, TST-031 (test REQ-012)
   - src/auth/service.py (code implementing DES-007)
   Shall I proceed and update the impacted artefacts?
   ```

7. **Coverage analysis** — The agent can generate coverage reports on request:
   - Requirements without tests
   - Designs without implementations
   - Decisions without traceability to requirements

## Examples

**Full frontmatter with traceability:**
```yaml
---
id: DES-007
artefact_type: design
title: "Authentication service component design"
sprint: 3
status: done
created: 2026-01-20
updated: 2026-01-22
author: pair
traces:
  derives_from: [REQ-010, REQ-011, REQ-012]
  decided_by: [DEC-003]
---
```
