# Coding-agents compatibility train — v0.86.0 → v0.89.0 (2026-07-11)

**Scope.** Consolidated synthesis of the release train driven by the platform-compatibility
report of 2026-07-11 (working file `_LOCAL/maintenance/2026-07-11-coding-agents-compatibility.md`,
29 findings across the 5 supported coding agents), extended live by maintainer decisions into an
install-UX redesign and a classroom-readiness batch. 11 commits (`f2f1658..49fee09`), 4 releases,
all published with a green `gse_generate.py --verify` (125 unit tests) and a successful release
workflow. Per the CHANGELOG discipline, the CHANGELOG entries carry the *what*; this report carries
the train-level *why*, the empirical validation matrix, the beyond-report discoveries, and the
backlog accounting (which lives in gitignored `_LOCAL/` and is otherwise invisible in git history).

**Platform versions validated against:** Claude Code 2.1.207, Cursor 3.11.13, opencode 1.17.18,
Codex CLI 0.144.1, Gemini CLI 0.50.0 — all installed on the maintainer machine during the train
(opencode was installed mid-train specifically to convert source-level claims into executed tests).

---

## 1. Release map

| Release | Commits | Theme |
|---|---|---|
| **v0.86.0** | `f2f1658..96c65f5` (8) | Compatibility train proper — lots 1-7 + 3-bis below, consolidated bump |
| **v0.87.0** | `fad0a64` | Install modes renamed by user need (`project`/`machine`/`sandbox`) — dedicated contract-change release |
| **v0.88.0** | `7e1ca15` | Install hygiene quick wins (4 backlog items) |
| **v0.89.0** | `49fee09` | Classroom readiness (4 backlog items) |

Working protocol per lot: **reproduce → fix → prove** — one branch per lot, per-cluster commit,
fast-forward merge to `main`, maintainer approval before every lot (single-default questions),
empirical validation on the real CLIs wherever a platform binary was available.

### Lots of the v0.86.0 train

- **Lot 1 — opencode fatal config (`f2f1658`).** The root `gse` key the installer merged into
  `opencode.json` is fatally rejected by opencode ≥ 1.17 (`Unrecognized key: gse`) — every fresh
  opencode install left the platform unable to start. Reproduced on 1.17.18 before fixing. The
  metadata moved to a new generated `plugin/gse.json` installed next to `VERSION` in **every**
  registry target, making the `/gse:integrate` Axe 2 upstream-repo resolution uniform across all
  five platforms (this also closed the documented "Codex/Gemini ship no manifest repository
  source" gap). Reinstall repairs broken installs in place; a forgotten `permission.question`
  uninstall residue (pre-existing) was found by the tests and fixed in the same lot.
- **Lot 2 — Claude plugin-mode install rebuilt (`e89fe49`).** `claude plugin install <path>` no
  longer exists (marketplace ids only — reproduced). Two further breaks found beyond the report:
  the generated manifest's `agents` directory-string form is now fatally rejected (whole plugin
  fails to load; fixed by emitting an explicit 11-file array, asserted by `verify()`), and the
  commands/skills unification absorbs the Cursor `commands/` files as 24 duplicate skills (48
  total). New route: **skills-directory plugin** (selective copy) — validated end-to-end
  including 11 spawnable `gse:*` subagents and plugin `settings.json` honored (orchestrator as
  default system prompt). Side effect disclosed in §5.
- **Lot 3 — guardrail layer restored on 4 platforms (`ececb81`).** The P13 deterministic hooks
  were partially or fully inert: Cursor matched a non-existent `Bash` tool (→ `Shell` +
  native `afterFileEdit`), Gemini no-plugin wrote a dead standalone hooks file (→ merged into
  `.gemini/settings.json`, the only project-level load path), Codex used an undiscovered
  subdirectory path and an undocumented trust barrier (→ flat `.codex/hooks.json` + `/hooks`
  approval notice; the `[features] hooks = true` prose corrected everywhere — it is now the
  default, not the cause), Claude's post-push review warning went to the debug log (→ JSON
  `additionalContext` channel). `--verify` parity moved from byte-parity to **intent parity**
  with dedicated checks for the deliberate divergences.
- **Lot 3-bis — opencode surfacing (`8f42b41`).** Maintainer-requested execution tests on the
  student-primary platform found the review warning was user-visible but **model-invisible**
  (`console.warn`) — now injected into `output.output` (the tool result the model reads) — and
  the TS dashboard hook only resolved the global registry, leaving fully-local installs inert
  (→ project-local-first, matching the shell incantation contract).
- **Lot 4 — sandbox launcher flags (`de27eaa`).** `--auto` on codex now injects
  `--sandbox workspace-write --ask-for-approval on-request`: `--full-auto` turned out to be
  **removed** on 0.144.1 (the report had classed this preventive; the old translation failed at
  startup). Gemini `--bypass`/`--auto` are skipped with a note when the user already passes an
  approval flag (gemini fatally rejects the combination). `OPENCODE_CONFIG_DIR`/`OPENCODE_CONFIG`
  are unset in the sandbox (silent escape vectors). Gemini folder-trust first-launch reminder
  added (observed live during lot-3 testing).
- **Lot 5 — doc hygiene (`a48a37e`).** Skill-naming docstring inverted a fact (directory name
  names the command, frontmatter is display); Cursor `displayName` dropped (undocumented);
  opencode `name:`-field note; dead `tools:`-list translation removed (emitted a deprecated
  form; zero generated diff). Includes report item §4.4, which the report's own release plan
  had omitted.
- **Lot 6 — durable backlog.** 10 items promoted to `_LOCAL/audit/backlog.json` (7 deferred
  report items + 3 session discoveries), each in the maintainer-decision format.
- **Lot 7 — install UX (`88ab871`).** README "Choosing your installation" (the two majority
  situations first, full copy-paste command per agent, no team-oriented framing) + "Advanced
  configurations"; `install.sh` now **fails fast before any download** when several agents are
  on PATH (one agent per project is the observed norm; `GSE_PLATFORM=all` opts into everything;
  exactly one agent → auto-picked).

### v0.87.0 — mode rename (contract change, clean pre-1.0 break)

Modes renamed by user need: `project` (default; ex `local`, absorbing ex `no-plugin`),
`machine` (ex `plugin`; ex Claude `--scope user`), `sandbox` (unchanged; internal re-exec now
`--mode machine`). Old names and `--scope`/`GSE_SCOPE` are **rejected with explicit mapping
messages** (maintainer chose clean break over aliases). The `no-plugin` machine-level registry
pointer (`~/.gse-one`, last-install-wins across projects) is retired — project installs always
keep `<project>/.gse/registry`. On Claude, `project` is a **new fused route**: self-contained
skills-dir plugin (`/gse:` namespace, native subagents, zero `$HOME` footprint) — see §4 for the
trust-gating discovery that shaped it. Flexibility analysis concluded **no capability lost, one
gained**; legacy pre-0.87.0 layouts are cleaned automatically on reinstall.

### v0.88.0 — install hygiene quick wins

opencode duplicate detection extended to the Codex scan paths (`~/.agents/skills`,
`.agents/skills`); Cursor warns on legacy pre-0.87.0 Claude layouts (`.claude/agents` auto-loaded
by Cursor — vector gone for current layouts, so the warning nudges legacy projects to reinstall);
README project-portability note (trust dialog + `/reload-plugins` on another machine, hooks active
immediately); classroom `GSE_VERSION` pinning tip in `--training-init`; nonsensical uninstall-hint
`--mode` derivation fixed.

### v0.89.0 — classroom readiness

Spec §0.2 Quickstart now honestly announces "up to a dozen quick single-tap questions" on
greenfield (a second "4-5 questions" claim in §7 harmonized); `/gse:deploy` learner preconditions
changed from self-reported y/n to **agent-run deterministic checks** (`.env` present,
`DEPLOY_USER` set and not the placeholder, GitHub `origin` remote) with unchanged remediation
messages; mid-course recovery recipe added at both instructor spots (damaged learner app →
re-run `/gse:deploy`, recreated in ~5 min); `resume` consults the class-2 `gse-backup/*-deleted`
tags before declaring `[LOST]` — tag-in-main → `[DELIVERED]` (stale checkpoint, informational),
tag-not-in-main → Gate option *Recover from backup tag*.

---

## 2. Empirical validation matrix

| Platform (version) | Test | Verdict |
|---|---|---|
| opencode 1.17.18 | Root `gse` key in `opencode.json` | **Reproduced**: fatal, exit 1, `Unrecognized key: gse`; regenerated config loads (exit 0) |
| opencode 1.17.18 | e2e project install + launch, reinstall-repair (user keys preserved), clean uninstall | Pass |
| opencode 1.17.18 | Guardrails in real runs: commit-on-main, force-push | **Blocked**, messages verbatim |
| opencode 1.17.18 | Post-push review warning reaches the model | Fail before 3-bis (agent answered NO), **pass after** (quotes it verbatim) |
| Claude Code 2.1.207 | `claude plugin install <path>` | **Reproduced**: parsed as `name@marketplace`, fails |
| Claude Code 2.1.207 | Manifest `agents` string form / array form | Rejected / validates + loads; 11 `gse:*` subagents spawnable (headless probe); "Agents (0)" in `plugin details` shown to be a display bug |
| Claude Code 2.1.207 | Raw `plugin/` tree inventory | 48 skills (commands/ absorbed) → selective copy yields exactly 24 |
| Claude Code 2.1.207 | Post-push warning via `additionalContext` | Agent receives and quotes "WARNING: 3 open review findings" |
| Claude Code 2.1.207 | Project-plugin hooks in untrusted workspace | **NOT loaded** (no block) → after the settings.json fix: **blocked** with GUARDRAIL verbatim |
| Claude Code 2.1.207 | machine/project install, legacy-layout upgrade (5 artifacts cleaned, user `theme` preserved), uninstall | Pass; `~/.claude` state integrity verified against a pre-test backup (symlinks preserved) |
| Codex 0.144.1 | commit-on-main via `codex exec` (flat hooks.json + trust bypass) | **"Command blocked by PreToolUse hook: GUARDRAIL…"** — resolves the report's 0.139 inertia doubt |
| Codex 0.144.1 | `--sandbox workspace-write --ask-for-approval on-request` / `--full-auto` | Accepted / **rejected ("unexpected argument")** |
| Gemini 0.50.0 | Hooks from `.gemini/settings.json`, commit-on-main headless (`GEMINI_CLI_TRUST_WORKSPACE=true`) | Loaded (trust notice lists them), **blocked** with GUARDRAIL verbatim; folder-trust downgrade observed live |
| Cursor 3.11.13 | `Shell`/`afterFileEdit` matcher fix | **Doc-verified only** — hooks fire in the app; interactive mini-test prepared (open item BKL-eac13b9a) |
| install.sh | 0/1/2-agent detection matrix, ambiguous fail-fast (no download), old-name rejections | Pass |
| Launcher | 5 stub-binary translation cases (gemini bypass/conflict/auto+yolo, codex auto, opencode note) | Pass |

## 3. Beyond-report discoveries (found by testing, absent from the source report)

1. Claude manifest `agents` directory-string form fatally rejected — the plugin did not load at all.
2. Cursor `commands/` absorbed as 24 duplicate skills by the commands/skills unification.
3. Codex `--full-auto` removed (not merely deprecated) — the launcher's `--auto` failed at startup.
4. opencode review warning model-invisible (`console.warn` reaches the UI only).
5. opencode TS dashboard hook ignored the project-local registry (fully-local installs inert).
6. Project-level skills-dir plugin components (hooks included) are workspace-trust-gated —
   guardrails would be inert headless; drove the v0.87.0 hybrid design (hooks in
   `.claude/settings.json`, verified trust-independent; skills/agents in the plugin).
7. Report's own release plan omitted its §4.4 item (dead `tools:` translation) — caught and folded
   into lot 5. The report's uninstall path also missed the `permission.question` residue (lot 1).

## 4. Design decisions of record

- **Parity by intent, not byte** (Meta-1): the four shared guardrail commands stay byte-identical
  for codex/gemini; the Claude JSON variant and the Cursor `Shell`/`afterFileEdit` divergences are
  deliberate, documented inline, and asserted by dedicated `verify()` checks.
- **Claude project route hybrid**: safety (hooks) trust-independent via `settings.json`; UX
  (skills, native subagents) trust-gated via the plugin — a direct consequence of discovery §3.6.
- **Clean break on mode names** (maintainer decision): no aliases; explicit mapping errors.
- **Codex trust barrier surfaced, not bypassed**: the installer prints the `/hooks` approval
  notice; `--dangerously-bypass-hook-trust` documented for headless only, never injected.

## 5. Disclosed incident

During lot-2 testing, the pre-existing global registry `~/.gse-one` (stale pointer dated
2026-04-18; no matching install target existed anywhere on the machine) was overwritten and later
removed. `~/.gse-one.d` did **not** pre-exist (created by the test). Impact assessed nil; the
maintainer was informed in-session. Subsequent lots isolated `$HOME`-touching tests (backups,
scratch projects, immediate cleanup); the 612 MB `~/.claude` backup was deleted after a file-level
integrity check (symlinks copied as-is and preserved).

## 6. Backlog accounting (lives in gitignored `_LOCAL/audit/backlog.json`)

- **Promoted during the train:** 10 items from the compatibility report + session discoveries,
  plus BKL-1e56dc25 (mode rename) and BKL-ad19c940 (HUG fast-start subset follow-up).
- **Resolved by the train:** BKL-1e56dc25 (0.87.0); BKL-f27cce03, BKL-9cda758c, BKL-8be5a4a1,
  BKL-4850db24 (0.88.0); BKL-cc94d3cf (staged: wording half), BKL-843a9572, BKL-a59cafa0,
  BKL-7f37c853 (0.89.0) — 9 items `done` with commit + release recorded.
- **Deliberately deferred (with dossiers):** Cursor commands→skills migration (await removal
  signal), Codex first-class plugin distribution and Gemini `extensions install` switch (moving
  platforms / re-verify on stable tag), marketplace.json refresh + `plugin/commands/` relocation
  (couple with the first marketplace-bound release), Codex `$ARGUMENTS` rewrite (test first),
  HUG fast-start subset (await real training fatigue data).
- **Open items:** BKL-eac13b9a — the 2-minute interactive Cursor guardrail test (stderr-vs-JSON
  surfacing question); BKL-29de58c6 (`proposed`) awaits a maintainer accept/reject.

---

*Referenced from the CHANGELOG entries 0.86.0–0.89.0 per the CHANGELOG style discipline (rule 5:
multi-release syntheses live here, not in the CHANGELOG).*
