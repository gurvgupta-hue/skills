---
name: secgraph
description: "Multi-stage agentic security scanning harness. Modelled on the MDASH pipeline (Prepare → Scan → Validate → Dedupe → Prove). Identifies hidden risks across CWE, OWASP Top 10, OWASP API Top 10, MITRE ATT&CK, MITRE CAPEC, CVE correlation, CVSS scoring, and zero-day heuristics. Deep supply-chain analysis: CycloneDX SBOM generation, Trivy CVE correlation, deps.dev enrichment, reachability analysis, EPSS exploitation-likelihood, CISA KEV known-exploited cross-reference, and a combined prioritisation score. Covers threat modelling (attack surface, trust boundaries, data flow, blast radius, privilege escalation, lateral movement), defensive analysis (auth, injection, API, secrets, cloud, containers, AI/LLM), offensive patterns (exploitability, business logic, supply chain), and code intelligence (SAST, SCA, taint analysis, call graph). Produces an executive dashboard (risk posture, KEV callouts, ranked 'fix these first' remediation, code-health score), a leadership one-pager, a full SBOM table (EPSS/reachability/priority as md + csv), PDF/Markdown reports, SARIF, and CycloneDX. Includes a code-quality auditor that runs real linters/formatters (Ruff, ESLint, Biome, PMD, Checkstyle, Stylelint, SwiftLint, Opengrep, and more) and normalises their output into a code-health rollup, kept separate from security findings. Handles small through monorepo-scale repos via adaptive strategy selection. Use when asked: analyse security, find vulnerabilities, threat model, scan for risks, OWASP audit, security review, identify attack paths, SBOM, supply-chain risk, executive security report, code quality, linting. Also runs an opt-in Joern Code Property Graph (CPG) taint/dataflow auditor for interprocedural source-to-sink analysis. Triggers also on: CPG, Joern, taint analysis, dataflow, source-to-sink, code quality, linting."
---

---

# SecGraph — Multi-Model Agentic Security Scanning Harness

> Modelled on the MDASH pipeline architecture: **Prepare → Scan → Validate → Dedupe → Prove**.
> The harness does the work. The model is one input.
> Every finding traces to a real file, line, and matched pattern. Nothing is fabricated.

---

## Table of Contents

Master routing file. Deep detail lives in `references/*.md`, loaded on demand.

1. Harness Architecture · 2. Anti-Hallucination Contract · 2A. Stage Announcements
3. Step 0 Prepare · 4. Step 1 Scan (→ `references/auditors.md`)
5. Validate · 7. Prove (→ `references/pipeline-detail.md`) · 6. Dedupe · 8. Threat Model · 9. Auto-Fix
10. Report (→ `references/reporting.md`) · 10A/10B Consolidate & Deliver
11. Scale (→ `references/scale-strategy.md`) · 12. Taxonomy (→ `references/taxonomy.md`) · 13. Plugins (→ `references/plugins.md`)
14. Scripts & references manifest

## 1. Harness Architecture

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    SECGRAPH AGENTIC HARNESS                             │
│                                                                         │
│  PREPARE      SCAN          VALIDATE      DEDUPE       PROVE            │
│  ┌───────┐   ┌──────────┐  ┌──────────┐  ┌────────┐  ┌─────────┐      │
│  │ Scale │──▶│ Auditor  │──▶│ Debater  │──▶│Cluster │──▶│ Prover  │     │
│  │ Triage│   │ Agents   │  │ Agents   │  │ & Rank │  │ Agents  │     │
│  │ Zone  │   │ (SAST,   │  │ (refute  │  │        │  │ (PoC,   │     │
│  │ Index │   │  secrets,│  │  or      │  │        │  │  chain) │     │
│  │ Trust │   │  API,IaC)│  │  confirm)│  │        │  │         │     │
│  └───────┘   └──────────┘  └──────────┘  └────────┘  └─────────┘     │
│                                                    │                    │
│                                              ┌─────▼──────┐            │
│                                              │  THREAT     │            │
│                                              │  MODEL +    │            │
│                                              │  REPORT     │            │
│                                              └────────────┘            │
└─────────────────────────────────────────────────────────────────────────┘
```

**Key architectural properties (from MDASH):**

- **Specialisation over generalisation.** Auditor agents find. Debater agents challenge. Prover agents confirm. No single agent does all three — that collapses the disagreement signal that makes the pipeline reliable.
- **Disagreement is a signal.** When an auditor flags something and a debater cannot refute it, posterior confidence rises. When a debater successfully refutes, the finding is downgraded — not dropped, but requeuable with new context.
- **The harness survives model changes.** Pattern files, zone configs, and plugin context are model-agnostic. Swapping the underlying model is a configuration change, not a rebuild.
- **Scale strategy is hardcoded into Prepare.** The harness decides how to execute based on repo size — not the operator. Guessing the wrong strategy for a 50,000-file monorepo produces noise, not findings.

---


## 2. Anti-Hallucination Contract

These rules apply at every stage. Violation of any rule means the finding must be suppressed, not downgraded.

**RULE 1 — Evidence required.** Every finding must cite: exact file path, exact line number, matched pattern text (the actual characters that triggered detection), and the pattern ID from `detection-patterns.json`. If any of these is missing, the finding is not emitted.

**RULE 2 — Taxonomy from source.** All CWE, OWASP, MITRE ATT&CK, CAPEC, and CVSS values come from `taxonomy-map.json` keyed by CWE ID. Never construct taxonomy references inline from training knowledge — taxonomy databases change and training data goes stale.

**RULE 3 — Unresolved stays unresolved.** A taint path that reaches a dangerous sink but cannot be fully traced from source to sink is labelled `taint_partial: true`. It is not upgraded to `confirmed` because you believe it is probably exploitable.

**RULE 4 — Zero-day heuristics are documented.** Zero-day classification is only applied when the finding matches a named indicator in `taxonomy-map.json → zero_day_indicators`. The indicator ID and signal are always cited. "This looks like a zero-day" without a matching documented heuristic is not a valid classification.

**RULE 5 — No CVE fabrication.** CVE IDs are never generated. Only CVEs explicitly referenced in a pattern's metadata are cited. Saying "this is similar to CVE-XXXX-YYYY" is only permitted when a pattern in `detection-patterns.json` explicitly maps to that CVE.

**RULE 6 — Scale gaps are disclosed.** When a repo scale strategy uses sampling (Zone-based or Multi-pass), the report's coverage section must state which zones were sampled, how many files per zone were read, and the estimated file coverage percentage. Never imply full coverage when sampling was used.

**RULE 7 — Confidence is always shown.** Every finding carries its confidence level: `critical` (pattern match is nearly unambiguous), `high` (strong pattern, low false-positive rate), `medium` (pattern is heuristic, manual review required), `low` (indicator only, verify before treating as vulnerability). Low and medium findings are never presented as confirmed without Prove stage confirmation.

**RULE 8 — Auto-fix suggestions come from the fix-pattern library, never from improvisation.** A fix suggestion is only generated when the finding's CWE ID has a matching template in `references/fix-patterns.json`, and it is always labelled `Suggested — verify before applying`. No suggestion is ever auto-applied to the repo. If no template matches, the harness states `no_template_available` rather than writing a plausible-looking but unverified fix.

**RULE 9 — Secret values are never written to any output, in any form.** `matched_text` for every `secrets`-category finding is redacted at the point of detection (`redact_secret_value()` in `scan_sast.py`) — first/last 2 characters visible, the rest masked, plus a one-way SHA-256 fingerprint (first 16 hex chars) used only to de-duplicate repeated occurrences of the same leaked value. This applies to `sast_findings.json`, `threat_model.json`, the markdown/PDF/HTML report, `results.sarif`, and autofix suggestions — no downstream stage re-reads the file to recover the raw value. If a future auditor or script needs the raw secret for verification (e.g. confirming a live credential during Prove), it may read the file directly for that one check but must never copy the raw value into any persisted finding record.

**RULE 10 — Exploitation signals are sourced, never estimated.** EPSS scores/percentiles, KEV membership, OpenSSF Scorecard values, and reachability classifications are only ever populated from a real source: the FIRST.org / CISA / deps.dev APIs, or a dated local cache file (`epss-scores.json`, `kev-catalog.json`). Claude never invents an EPSS number, never asserts a CVE is or isn't in KEV from memory, and never upgrades a `reachability` verdict from `undetermined` to `reachable` without an actual matched call site. When any of these signals is missing, the field is written as `"unavailable"` (or `undetermined` for reachability) and the derived `priority_decision` records that a likelihood signal was absent — a false-confident low priority is a Rule 10 violation. The `priority_score` is a documented deterministic function of these inputs (see `PRIORITY_WEIGHTS` in `enrich_sbom.py`), not a model judgement, so any reader can reproduce it.

---


## 2A. Stage Progress Announcements (Required)

Every pipeline stage is bracketed by a printed header and footer so the operator can see exactly where the run is, even on long/monorepo scans. **This is required output, not optional narration** — print it in the chat/log, not just in files.

**Start-of-stage format:**
```
════════════════════════════════════════════════════════
▶ STARTED — <Stage Name>
════════════════════════════════════════════════════════
```

**End-of-stage format:**
```
────────────────────────────────────────────────────────
✔ FINISHED — <Stage Name>   (<n> findings / <n> items — <elapsed if known>)
────────────────────────────────────────────────────────
```

Use the canonical stage names below — do not invent variants, so downstream log-scraping and the final consolidated view can key off them reliably:

| # | Stage Name |
|---|---|
| 0 | `Prepare — Repo Intake & Scale Strategy` |
| 1a | `Scan — SAST / Injection & Dangerous Sinks` |
| 1b | `Scan — Secrets & Credential Detection` |
| 1c | `Scan — API Security` |
| 1d | `Scan — Infrastructure & Container Security` |
| 1e | `Scan — SBOM / CVE / Reachability Analysis` |
| 1f | `Scan — AI/LLM Security` |
| 1g | `Scan — CI Workflow Security` |
| 1h | `Scan — Mobile Security` |
| 2 | `Validate — Debater Review` |
| 3 | `Dedupe — Semantic Deduplication` |
| 4 | `Prove — Exploitability Confirmation` |
| 5 | `Threat Model Synthesis` |
| 6 | `Auto-Fix Suggestion Generation` |
| 7 | `Report Generation (Markdown / SARIF / PDF)` |

A stage's footer must report a real count (findings emitted, files scanned, dependencies inventoried, etc.) — never a placeholder. If a stage is skipped entirely by the scale strategy (e.g. Prove skipped for Tier-3 files at Medium scale), print `⤫ SKIPPED — <Stage Name> (reason: ...)` instead of a start/finish pair, so the operator knows it wasn't silently forgotten.

At the very end of the run, print one **Consolidated Run Summary** block that lists every stage with its status (✔ finished / ⤫ skipped) and count, immediately before the final report links — see [Section 9A](#9a-consolidated-view--final-outputs).

---


## 3. Step 0 — Prepare: Repo Intake and Scale Strategy

> **Announce:** `▶ STARTED — Prepare — Repo Intake & Scale Strategy`

### Step 0.1 — Locate the repository

```bash
# If given a GitHub URL:
git clone --depth 1 <url> /home/claude/repo_workspace/<repo-name>

# If already local, use the path directly.
# If private/auth-required and inaccessible, say so — never fabricate findings
# from the repo name or description alone.
```

### Step 0.2 — File census

```bash
python scripts/scan_sast.py --census-only \
  --root /home/claude/repo_workspace/<repo-name> \
  --out /home/claude/secout/census.json
```

The census emits: total file count, source file count, LOC estimate, language breakdown, top-level directory list, and presence of package manifests / IaC / pipeline files.

### Step 0.3 — Build the zone index

A **zone** is a semantically coherent directory unit — not an arbitrary file batch. Zone boundaries come from:
1. Top-level directories (always)
2. Package manifest locations (`package.json`, `pyproject.toml`, `go.mod`, etc.) — each manifest root is a zone boundary
3. IaC directories (`infra/`, `terraform/`, `.azure/`, `bicep/`) — always a separate zone
4. Pipeline directories (`.github/workflows/`, `.azure-pipelines/`) — always a separate zone
5. `tests/`, `spec/`, `__tests__/` — always a separate zone (lower priority, scan last)

Emit the zone list before selecting strategy. Ask the user to confirm if zone count > 10 or if zone boundaries look wrong.

**Monorepo primary-project selection.** If Step 0.2's census shows multiple independent manifest roots (multiple `package.json`/`pom.xml`/`go.mod` in separate directories, `/apps/`, `/services/`, `/packages/` structure, or workspace tooling like Nx/Turborepo/Bazel/Lerna), this is a monorepo of genuinely separate projects, not just a large single codebase. In that case:
- Ask the user which project(s) are in scope, or infer the primary one from what was asked about (e.g. "audit the payment service" → that's the primary project even if five others live alongside it)
- Give every non-selected project a 2–3 sentence scope note (what it is, why it's out of scope for this pass) instead of scanning it at reduced depth — a shallow scan of an out-of-scope project produces noise, not signal
- Each in-scope project still goes through full zone indexing and strategy selection independently; a monorepo with one 300-file service and one 8,000-file service should not force the small service into Zone-based sampling

### Step 0.3A — Context Budget Allocation

Before deep scanning starts, allocate the audit's effort across stages so heavy analysis (Prove, Threat Model) isn't starved by an over-long Prepare/Scan phase on large repos. Treat this as a soft budget — a guide for where to spend manual reading time and how deep to go, not a hard token cap the harness enforces programmatically:

| Stage | Budget Share (Large+) | Notes |
|---|---|---|
| Prepare (Step 0) | ~10% | Census, zone index, manifests, README — cheap, high-value |
| Scan (Step 1, all auditors) | ~40% | Script-driven auditors (SAST/SCA) cover 100% regardless of budget; manual auditor reading (API, IaC review, LLM) is what gets sampled under this budget |
| Validate + Dedupe (Steps 2–3) | ~10% | Mostly pattern-based; cheap relative to Scan |
| Prove (Step 4) | ~20% | Reserve real budget here — this is where findings become vulnerabilities. Don't let Scan sampling starve Prove's depth on the findings that did surface |
| Threat Model + Auto-Fix (Steps 5–6) | ~10% | Synthesis over what's already found — should not require re-reading the repo |
| Report + Deliver (Steps 7–8) | ~10% | Rendering, not analysis |

**Budget enforcement rules (apply throughout Scan and Prove on Large+ strategies):**
- Never read an entire large file end-to-end during manual auditor reading. Read the first 100–150 lines plus a targeted grep/search for the pattern in question; expand only if the finding needs more context to evaluate.
- Prefer breadth over depth when sampling — one representative file per module beats reading every file in one module and none in the next.
- Stop reading a category once it's sufficiently understood. If three files in a zone confirm the same auth pattern, a fourth confirming file doesn't earn its context cost.
- Log every skip. Maintain `/home/claude/secout/skipped_areas.json` — see below.

**Skipped Areas Log.** For every directory, zone, or file category not examined (or examined only via script, not manual read) under a sampling strategy, record an entry:
```json
{
  "path": "services/legacy-billing/",
  "reason": "Tier 3 sampling — zone budget exhausted after Tier 1/2 files",
  "likely_contents": "Older billing reconciliation logic, per directory naming and manifest",
  "potential_impact_if_examined": "medium — handles payment data but zone summary shows no direct external HTTP exposure"
}
```
This log feeds directly into the report's Coverage disclosure (Section 10, Step 7) and into the Self-Reflection Checkpoint below — a skip with `potential_impact_if_examined: "high"` should be flagged as a follow-up priority, not buried.

### Step 0.4 — Audit Strategy Selection

Select strategy based on source file count (not total file count — ignore assets, docs, and generated files):

| Repo Scale | Source Files | Strategy | Description |
|---|---|---|---|
| **Small** | < 500 | **Full scan** | Read every source file. No sampling. Run complete pipeline on all files. |
| **Medium** | 500–2,000 | **Standard v2.0** | Full SAST on all source files. Prioritise files by: entry points first, then files touching auth/crypto/network, then the rest. |
| **Large** | 2,000–5,000 | **Hierarchical summarisation** | Bottom-up: summarise each zone's structure and key files before synthesising cross-zone threat model. Run SAST on full file set but build threat model zone-by-zone. |
| **Very Large** | 5,000–10,000 | **Zone-based sampling** | Divide into 3–7 zones. Per zone: read README/index + manifest + 3–5 representative files (entry points, auth, core logic). Synthesise zone summary before proceeding. Full SAST still runs via script; manual reading is sampled. |
| **Massive / Monorepo** | 10,000+ or > 100 MB | **Multi-pass sequential** | Pass 0: triage only — census, zone map, file priority tiers. Report zones and request user confirmation of deep-dive zones before Pass 1. Pass 1: full SAST script on confirmed zones only. Pass 2+: per-zone deep dives on request. |

**Strategy is selected once at Step 0.4 and does not change mid-audit.** If the strategy needs to change (e.g., a zone turns out to be much larger than expected), surface this to the user and request confirmation before switching — never silently switch to a cheaper strategy.

### Step 0.5 — Zone-based sampling execution (Very Large / Massive only)

For each zone, in priority order:

```
Zone sampling protocol per zone:
1. Read README.md or index file if present (max 200 lines)
2. Read package manifest / build file (all of it)
3. Identify entry points:
   - Web: routes files, controllers, app.py/server.js/Program.cs equivalents
   - CLI: main() / __main__ / entry_point declarations
   - API: OpenAPI/Swagger specs, router files
   - IaC: top-level .tf / main.bicep / azuredeploy.json
4. Read 3–5 representative files — prioritise by:
   a. Files touching authentication, authorisation, or session management
   b. Files handling external input (HTTP, file upload, message queues)
   c. Files with cryptographic operations
   d. Core business logic files (highest cyclomatic complexity if detectable)
5. Write a zone summary (3–5 sentences):
   - What the zone does
   - What external inputs it accepts
   - What trust boundaries it crosses
   - What security-relevant patterns were observed
   - What warrants deeper investigation
```

**Do not proceed to Step 1 (Scan) until all zone summaries are written.** The zone summaries are the input to the cross-zone threat model in Step 5.

### Step 0.6 — Install dependencies

```bash
bash scripts/setup_env.sh
```

Reports which tree-sitter grammars installed successfully. Files in unsupported languages are excluded and disclosed in the final report's coverage section.

---

> **Announce on completion:** `✔ FINISHED — Prepare — Repo Intake & Scale Strategy   (<n> files censused, strategy=<Small|Medium|Large|Very Large|Massive>)`

---


## 4. Step 1 — Scan: Specialised Auditor Agents

> **Announce:** `▶ STARTED — Step 1 Scan`

Run each auditor in turn over the in-scope files. Each writes findings to its own JSON as `{id, category, cwe, file, line, matched_text, confidence, severity}`, every entry traced to a real file/line (Rule 1). Auditors, in order:

| Auditor | Announce as | Covers |
|---|---|---|
| A | `Scan — SAST` | injection & dangerous sinks (tree-sitter/regex) |
| B | `Scan — Secrets` | credentials & key material |
| C | `Scan — API Security` | authz, BOLA, mass-assignment |
| D | `Scan — IaC` | Dockerfile / Terraform / Kubernetes |
| E | `Scan — SBOM / CVE / Reachability / EPSS` | supply chain (E.1–E.7) |
| F | `Scan — AI/LLM Security` | prompt injection, unsafe tool use (if applicable) |
| G | `Scan — CI Workflow Security` | GitHub Actions / pipelines |
| H | `Scan — Mobile Security` | Android / iOS (if applicable) |
| Q | `Scan — Code Quality & Linting` | real linters/formatters → code-health (separate from security) |
| T | `Scan — Taint / Dataflow (CPG)` | **opt-in, heavy** Joern CPG source-to-sink; run only on explicit request |

**Before running any auditor, read its full rule set in `references/auditors.md`** — that file holds the exact patterns, per-language rules, the E.1–E.7 supply-chain detail (Trivy/deps.dev/EPSS/KEV/reachability/priority), and the opt-in Auditor T (CPG) contract. Auditor T is off by default; run it only when the user asks for CPG/taint/dataflow.

> **Announce on completion:** `✔ FINISHED — Step 1 Scan (all auditors)   (<n> raw findings across A–H + Q [+ T if run])`

## 5. Step 2 — Validate: Debater Agents

> **Announce:** `▶ STARTED — Validate — Debater Review`

A debater argues against each auditor finding. A finding survives only if no reasonable rebuttal exists; a rebuttable finding is **downgraded in confidence, not dropped** (unless the rebuttal is definitive). **Read `references/pipeline-detail.md` § Validate for the per-class debater protocols (injection, secrets, auth, IaC) and the required output format before running this stage.**

> **Announce on completion:** `✔ FINISHED — Validate`

## 6. Step 3 — Dedupe: Semantic Deduplication

> **Announce:** `▶ STARTED — Dedupe — Semantic Deduplication`

Before building the threat model, collapse findings that represent the same root vulnerability:

**Deduplication rules:**
1. Same `(file, line, pattern_id)` → exact duplicate → keep one, note count
2. Same vulnerability in same function across different pattern matches → merge into one finding, keep highest CVSS
3. Same vulnerability class in same file, lines within 5 of each other → likely same code path → merge, cite both lines
4. Cross-file: same vulnerability in multiple files of same type (e.g., 12 routes all missing auth) → keep individual entries in detailed findings but create one aggregate entry in the executive summary to avoid report noise

**Deduplication is structural, not semantic.** Do not merge a SQL injection in `routes/users.py` with one in `routes/orders.py` just because they are "both SQL injections" — they are separate attack surface nodes.

> **Announce on completion:** `✔ FINISHED — Dedupe — Semantic Deduplication   (<n> findings collapsed into <n> unique)`

---


## 7. Step 4 — Prove: Exploitability Confirmation

> **Announce:** `▶ STARTED — Prove — Exploitability Confirmation`

Attempt to confirm each surviving finding is genuinely exploitable, with concrete evidence, raising or lowering confidence accordingly. A correct earlier refusal is never reversed by an emotional appeal or a prior-session summary. **Read `references/pipeline-detail.md` § Prove for the per-class proof protocols (injection, secrets, IaC/container) before running this stage.**

> **Announce on completion:** `✔ FINISHED — Prove`

## 8. Step 5 — Threat Model Synthesis

> **Announce:** `▶ STARTED — Threat Model Synthesis`

### Self-Reflection Checkpoint (required on Large+ strategies, optional but encouraged on Small/Medium)

Before running the threat model builder, pause and interrogate the run so far — this is cheap relative to the cost of a threat model built on unexamined assumptions:

1. **List at least 3 assumptions** made during Scan/Prove — about architecture (e.g. "assumed all routes go through the central auth middleware because Tier 1 files did"), about deployment (e.g. "assumed this is the production Dockerfile"), or about zone/project boundaries.
2. **Challenge each one:** How could it be wrong? What evidence would disprove it? Did a skipped area (from `skipped_areas.json`) plausibly contain disconfirming evidence?
3. **Assess coverage honestly:** what percentage of source files were actually read (manual auditor reads, not just SAST-script-parsed), by zone and by functional area. This number feeds the report's Coverage disclosure verbatim — don't round it up.
4. **Identify the single largest blind spot** — the zone or file category with the least evidence that could most change the threat model's conclusions if examined. Surface it explicitly as a follow-up priority, not a footnote.
5. **Mark any finding built on a challenged assumption as `tentative: true`** in its record, and carry that flag into the threat model and report — a tentative critical finding should still be reported as critical, but visibly flagged for verification before it's treated as gospel in a remediation sprint.

Run after Prove stage. Inputs: validated + deduped + proven findings, zone summaries (if applicable).

```bash
python scripts/build_threat_model.py \
  --sast /home/claude/secout/sast_findings.json \
  --taxonomy references/taxonomy-map.json \
  --out /home/claude/secout/threat_model.json
```

The threat model builder computes:
- **Attack surface nodes** — entry points categorised by type
- **Trust boundary crossings** — where untrusted data enters trusted systems
- **Attack paths** — multi-step chains evidenced by actual findings
- **Privilege escalation paths** — findings that enable PrivEsc (container escape, credential theft, auth bypass chained)
- **Lateral movement potential** — findings that, if exploited, enable movement across services
- **Blast radius** — per finding, structural estimate of propagation scope
- **CVSS-adjusted risk scores** — base CVSS × confidence multiplier + lateral/privesc bonuses
- **Zero-day candidates** — findings matching documented heuristics in taxonomy-map.json
- **Control coverage gaps** — OWASP Top 10 / API Top 10 categories with no findings (verify vs gap)

**Cross-zone threat synthesis (Large+ strategies only):**
After zone-level threat models are built, synthesise across zones:
- Which findings in Zone A enable attack surface expansion into Zone B?
- Which credential findings in one zone enable lateral movement to another?
- Are there trust boundary crossings between zones that were not obvious from single-zone analysis?
This synthesis is the primary value of multi-pass analysis vs single-zone — explicitly surface cross-zone attack paths.

> **Announce on completion:** `✔ FINISHED — Threat Model Synthesis   (<n> attack paths, <n> attack surface nodes, <n> zero-day candidates)`

---


## 9. Step 6 — Auto-Fix Suggestion Generation

> **Announce:** `▶ STARTED — Auto-Fix Suggestion Generation`

**This stage suggests. It never edits the repo.** No file in `repo_workspace/` is modified by this stage under any circumstance — output is patch-style suggestions only, for a human (or a separate, explicitly-approved apply step) to review and apply.

For every finding at `confidence=critical` or `confidence=high` that survived Dedupe (Prove result of `proven` or `partial`), attempt to generate a fix suggestion:

```bash
python scripts/suggest_autofix.py \
  --threat-model /home/claude/secout/threat_model.json \
  --fix-patterns references/fix-patterns.json \
  --repo-root /home/claude/repo_workspace/<repo-name> \
  --out /home/claude/secout/autofix_suggestions.json
```

**Anti-hallucination Rule 8 — Fixes come from the fix-pattern library, never from general coding instinct.** Look up the finding's CWE ID in `references/fix-patterns.json`. If a template exists for that CWE:
- Render the template's before/after snippet against the finding's actual file/line/matched text
- Label it clearly: `⚠ Suggested — verify and test before applying`
- Include the fix-pattern ID cited, same as taxonomy citations elsewhere

If no template exists for that CWE, do **not** improvise one. Emit `"autofix": "no_template_available"` with a one-line pointer to the relevant OWASP cheat-sheet category (name only, not fabricated code) and move on. A missing suggestion is honest; a fabricated one is a liability.

**What the suggestion is, and is not:**
- It IS a targeted, minimal diff addressing the specific matched pattern (e.g. parameterising the one query, moving the one secret to an env var reference)
- It is NOT a refactor, NOT a rewrite of surrounding logic, and NOT guaranteed to compile/run without adjustment — dependencies, framework conventions, and test coverage vary per repo
- Suggestions for `low`/`medium` confidence or `unproven` findings are not generated — fix effort should follow proof strength

Output format per suggestion (in `autofix_suggestions.json`, one entry per finding):
```json
{
  "finding_id": "...",
  "cwe": "CWE-89",
  "file": "routes/users.py",
  "line": 42,
  "fix_pattern_id": "FIX-CWE-89-001",
  "diff": "- query = f\"SELECT * FROM users WHERE id = {user_id}\"\n+ query = \"SELECT * FROM users WHERE id = %s\"\n+ cursor.execute(query, (user_id,))",
  "confidence_in_suggestion": "template_match | partial_match",
  "note": "Verify parameter binding syntax matches the driver in use (this repo uses psycopg2)."
}
```

> **Announce on completion:** `✔ FINISHED — Auto-Fix Suggestion Generation   (<n> suggested, <n> no-template)`

---


## 10. Step 7 — Report Generation (Markdown / SARIF / PDF / HTML)

> **Announce:** `▶ STARTED — Report Generation (Markdown / SARIF / PDF / HTML)`

Prerequisite: `sbom_enriched.json` must already carry reachability + EPSS + KEV + priority (Auditor E.3/E.5/E.6/E.7). If the reachability pass was skipped, run it now so the executive view can scope priority correctly:

```bash
python scripts/analyze_reachability.py \
  --root /home/claude/repo_workspace/<repo-name> \
  --sbom-enriched /home/claude/secout/sbom_enriched.json \
  --out /home/claude/secout/sbom_enriched.json
```

```bash
# render_report.py now takes --sbom so the markdown/PDF gets the supply-chain
# executive section AND a standalone executive_summary.md one-pager.
python scripts/render_report.py \
  --threat-model /home/claude/secout/threat_model.json \
  --taxonomy references/taxonomy-map.json \
  --sbom /home/claude/secout/sbom_enriched.json \
  --quality /home/claude/secout/quality_findings.json \
  --quality-summary /home/claude/secout/quality_summary.json \
  --autofix /home/claude/secout/autofix_suggestions.json \
  --out-dir /home/claude/secout/reports

python scripts/export_sarif.py \
  --threat-model /home/claude/secout/threat_model.json \
  --taxonomy references/taxonomy-map.json \
  --out /home/claude/secout/reports/results.sarif

python scripts/render_pdf.py \
  --md /home/claude/secout/reports/security_report.md \
  --out /home/claude/secout/reports/security_report.pdf

# The HTML dashboard's Executive Overview is driven entirely by sbom_enriched.json's
# priority/EPSS/KEV fields — pass it, or the exec cards render as zeros.
python scripts/render_dashboard.py \
  --threat-model /home/claude/secout/threat_model.json \
  --sbom /home/claude/secout/sbom_enriched.json \
  --quality /home/claude/secout/quality_findings.json \
  --quality-summary /home/claude/secout/quality_summary.json \
  --autofix /home/claude/secout/autofix_suggestions.json \
  --skipped-areas /home/claude/secout/skipped_areas.json \
  --out /home/claude/secout/reports/security_dashboard.html
```


**Full report / dashboard / executive-view spec — the outputs table, why the dashboard is separate from the PDF/MD report, evidence tiers & confidence calibration, the final self-critique gate, and the mandatory coverage-disclosure section — is in `references/reporting.md`. Read it before finalising the report.**

## 10A. Consolidated View — Final Outputs

Immediately after the Report Generation footer, print one consolidated block before handing control back to the operator. This is the "we already have a view, plus a PDF and SARIF" deliverable the pipeline builds toward:

```
════════════════════════════════════════════════════════
  CONSOLIDATED RUN SUMMARY — <repo-name>
════════════════════════════════════════════════════════
  Strategy:            <Small|Medium|Large|Very Large|Massive>
  Coverage:             <n>% (<n>/<n> files)

  ✔ Prepare                                    <n> files censused
  ✔ Scan — SAST                                <n> findings
  ✔ Scan — Secrets                             <n> findings
  ✔ Scan — API Security                        <n> findings
  ✔ Scan — Infrastructure/Container (Dockerfile/Terraform/K8s)   <n> findings
  ✔ Scan — SBOM/CVE/Reachability/EPSS/KEV       <n> deps, <n> CVE-correlated, <n> reachable, <n> high-EPSS, <n> KEV, <n> act-now
  ✔ Scan — AI/LLM Security                     <n> findings   (⤫ skipped if N/A)
  ✔ Scan — Code Quality & Linting              <n> lint, <n> format, health <n>/100   (⤫ tools disclosed if missing)
  ✔ Scan — CI Workflow Security                 <n> findings   (⤫ skipped if no workflows)
  ✔ Scan — Mobile Security (Android/iOS)         <n> findings   (⤫ skipped if no mobile project)
  ✔ Validate — Debater Review                   <n> confirmed / <n> downgraded / <n> refuted
  ✔ Dedupe                                      <n> unique findings
  ✔ Prove                                       <n> proven / <n> partial / <n> unproven
  ✔ Threat Model Synthesis                      <n> attack paths, <n> zero-day candidates
  ✔ Auto-Fix Suggestions                        <n> suggested / <n> no-template
  ✔ Report Generation                           md + exec one-pager + pdf + html dashboard + sarif + cyclonedx + sbom_enriched emitted

  Findings:  🔴 <n>  🟠 <n>  🟡 <n>  🟢 <n>
  Supply chain:  <n> KEV (known-exploited)  ·  <n> act-now  ·  <n> review-manually (no signal)
  Code health:   <n>/100  ·  formatting <n>% compliant  ·  <n> lint findings

  Reports:
    - /home/claude/secout/reports/executive_summary.md   ← leadership one-pager
    - /home/claude/secout/reports/security_report.md
    - /home/claude/secout/reports/security_report.pdf
    - /home/claude/secout/reports/security_dashboard.html ← exec overview + full data
    - /home/claude/secout/reports/results.sarif
    - /home/claude/secout/reports/autofix_suggestions.md
    - /home/claude/secout/sbom.cdx.json                   ← CycloneDX SBOM
    - /home/claude/secout/reports/sbom_table.md / .csv       ← full SBOM table view
    - /home/claude/secout/sbom_enriched.json
    - /home/claude/secout/quality_summary.json               ← code-health rollup
════════════════════════════════════════════════════════
```

Every count in this block must be a real number pulled from the stage outputs — never estimated or left as a placeholder in the actual run.

---


## 10B. Step 8 — Deliver Outputs

> **Announce:** `▶ STARTED — Deliver Outputs`

The `/home/claude/secout/` tree is inside the scanning sandbox — it is **not** visible to the operator on its own. This stage is what actually hands the deliverables over, and it is required, not optional, whenever this harness is run from within an environment that has an operator-facing outputs directory (e.g. Claude's `/mnt/user-data/outputs/`).

```bash
mkdir -p /mnt/user-data/outputs/<repo-name>
cp /home/claude/secout/reports/executive_summary.md   /mnt/user-data/outputs/<repo-name>/
cp /home/claude/secout/reports/security_report.md      /mnt/user-data/outputs/<repo-name>/
cp /home/claude/secout/reports/security_report.pdf     /mnt/user-data/outputs/<repo-name>/
cp /home/claude/secout/reports/security_dashboard.html /mnt/user-data/outputs/<repo-name>/
cp /home/claude/secout/reports/results.sarif           /mnt/user-data/outputs/<repo-name>/
cp /home/claude/secout/reports/autofix_suggestions.md  /mnt/user-data/outputs/<repo-name>/
cp /home/claude/secout/sbom.cdx.json                   /mnt/user-data/outputs/<repo-name>/
cp /home/claude/secout/sbom_enriched.json              /mnt/user-data/outputs/<repo-name>/
cp /home/claude/secout/reports/sbom_table.md           /mnt/user-data/outputs/<repo-name>/ 2>/dev/null || true
cp /home/claude/secout/reports/sbom_table.csv          /mnt/user-data/outputs/<repo-name>/ 2>/dev/null || true
cp /home/claude/secout/quality_summary.json            /mnt/user-data/outputs/<repo-name>/ 2>/dev/null || true
cp /home/claude/secout/reports/findings_critical.md    /mnt/user-data/outputs/<repo-name>/
cp /home/claude/secout/reports/findings_high.md        /mnt/user-data/outputs/<repo-name>/
cp /home/claude/secout/reports/attack_surface.mmd      /mnt/user-data/outputs/<repo-name>/
```

Then present the files to the operator (in Claude's environment, call `present_files` on the copied paths — lead with `security_dashboard.html` or `security_report.pdf` as the primary deliverable, since `present_files` treats the first path as the one most relevant to show first). Do not just print the sandbox paths and stop — a path the operator can't reach is not a deliverable.

> **Announce on completion:** `✔ FINISHED — Deliver Outputs   (<n> files copied to /mnt/user-data/outputs/<repo-name>/)`

---


## 11. Repo Scale Strategy Matrix

Pick a strategy from the Step 0 file census:

| Strategy | Source files | Approach |
|---|---|---|
| Small | < 500 | all auditors on the full repo |
| Medium | 500–2,000 | full scan; batch the heavy auditors |
| Large | 2,000–5,000 | zone-prioritised; large-repo tuning |
| Very Large | 5,000–10,000 | zone sampling + deep-dive follow-up |
| Massive / Monorepo | 10,000+ or > 100 MB | sampled zones; compact mode if context-bound |

**Read `references/scale-strategy.md` for per-scale execution detail, large-repo tuning, the zone deep-dive follow-up, and the compact-mode fallback.**

## 12. Taxonomy Reference

Findings map to CWE 4.14, OWASP Top 10 2021, OWASP API 2023, OWASP Mobile 2024, MITRE ATT&CK v14, CAPEC 3.9, CVSS v3.1, EPSS, and CISA KEV. **Full coverage tables, CVSS severity bands / remediation SLAs, and zero-day heuristics are in `references/taxonomy.md`**; the machine-readable map is `references/taxonomy-map.json`.

## 13. Plugin Extensibility

Auditors and rule packs can be extended via a plugin interface. **See `references/plugins.md` for the interface, placement, and example plugin scenarios.**

## 14. Scripts & References Manifest

All executables live in `scripts/`, all data and detail in `references/`. Invoke scripts as `python scripts/<name>` / `bash scripts/<name>`. Files stay on disk at zero context cost until a step needs them.

### scripts/
| Script | Step | Purpose |
|---|---|---|
| `setup_env.sh` | Step 0 | install tree-sitter grammars + certifi TLS store; probe FIRST.org |
| `scan_sast.py` | Auditor A | tree-sitter/regex SAST |
| `scan_sca.py` | Auditor E.1 | SBOM + valid CycloneDX |
| `enrich_sbom.py` | Auditor E.2–E.7 | Trivy CVEs + deps.dev + EPSS + KEV + priority |
| `analyze_reachability.py` | Auditor E.3 | import/symbol reachability + finalise priority |
| `scan_quality.py` | Auditor Q | linters/formatters → code-health rollup |
| `scan_cpg.py` (+ `check_env_cpg.sh`, `build_cpg.sh`, `run_scan.sh`, `run_scan.sc`) | Auditor T (opt-in) | Joern CPG taint/dataflow |
| `build_threat_model.py` | Step 5 | synthesise threat model / risk register |
| `suggest_autofix.py` | Step 6 | auto-fix suggestions |
| `render_report.py` | Step 7 | md report + exec one-pager + SBOM table (md/csv) |
| `render_dashboard.py` | Step 7 | interactive HTML dashboard (exec overview) |
| `export_sarif.py` | Step 7 | SARIF 2.1.0 |
| `render_pdf.py` | Step 7 | PDF from the md report |
| `install_trivy.sh`, `install_quality_tools.sh` | setup | optional tool installers |

### references/
| File | Loaded by | Contents |
|---|---|---|
| `auditors.md` | §4 | full auditor A–H, Q, T rules |
| `pipeline-detail.md` | §5, §7 | Validate & Prove protocols |
| `reporting.md` | §10 | report/dashboard/exec spec, evidence tiers, coverage |
| `scale-strategy.md` | §11 | per-scale tuning, compact mode |
| `taxonomy.md` | §12 | taxonomy tables, SLAs, zero-day heuristics |
| `plugins.md` | §13 | plugin interface |
| `cpg-query-catalog.md` | Auditor T | Joern query catalog + taxonomy mapping |
| `detection-patterns.json` | Auditors A–D,G,H | detection pattern library |
| `fix-patterns.json` | Step 6 | auto-fix templates |
| `taxonomy-map.json` | Step 5/7 | CWE/OWASP/ATT&CK map |
| `approved-base-images.json`, `secureflow-templates.json` | Auditor D | IaC baselines |
| `epss-scores.json`, `kev-catalog.json` | Auditor E.5/E.6 | offline EPSS / KEV caches |

---

*SecGraph — Multi-Model Agentic Security Scanning Harness.*
*Pipeline: Prepare → Scan (SAST/Secrets/API/IaC/SBOM+CVE+Reachability+EPSS+KEV+Priority/LLM/CI Workflows/Mobile/Code-Quality) → Validate → Dedupe → Prove → Threat Model → Auto-Fix Suggestions → Report (Executive one-pager + full SBOM table + MD/PDF/SARIF/HTML dashboard + CycloneDX) → Deliver.*
*Languages: Python, JavaScript, TypeScript, Vue, Java, Kotlin, Swift, Objective-C, Go, C#, PHP, Ruby, C/C++, Rust (SCA only), Terraform, Bicep, Kubernetes YAML.*
*Taxonomy: CWE 4.14 | OWASP Top 10 2021 | OWASP API 2023 | OWASP Mobile Top 10 2024 | MITRE ATT&CK v14 | CAPEC 3.9 | CVSS v3.1 | EPSS | CISA KEV.*
*Code quality (Auditor Q): PMD | Checkstyle | google-java-format | Spotless | ESLint | Prettier | Biome | Ruff | Stylelint | isort | flake8 | SwiftLint | swift-format | Opengrep.*
