# save-yourself v0.3 Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Refactor save-yourself into a two-tier orchestrator that dispatches parallel Phase 3 agents per stack, adds flow diagrams/hard gates to SKILL.md, and improves Java vulnerability detection with CycloneDX SBOM generation.

**Architecture:** SKILL.md becomes the orchestrator: Phases 0–2 and CP2 run sequentially in the main session, a pre-dispatch tool check offers to install missing tools interactively, Phase 3 spawns one parallel Agent per detected stack (each writing normalized JSON to `.claude/save-yourself-audit-<stack>.json`), and Phase 4 merges all JSON files into the final report.

**Tech Stack:** Markdown skill files, Claude Code Agent tool, CycloneDX Maven plugin (`org.cyclonedx:cyclonedx-maven-plugin:2.7.9`), osv-scanner `--sbom` flag, npm audit, govulncheck, cargo audit, pip-audit

---

### Task 1: Add "When NOT to use" guard and DOT flow diagram to SKILL.md

**Files:**
- Modify: `skills/save-yourself/SKILL.md`

- [ ] **Step 1: Add "When NOT to use" section**

In `SKILL.md`, after the line:
```
Before starting, tell the user: "Running /save-yourself security scan. This will
check your .env setup, scan for leaked credentials, and audit dependencies. I'll
narrate each step as I go."
```

Insert this block:

```
## When NOT to use this skill

- User only wants to check .env: run Phase 2 only, skip Phase 3 entirely.
- No dependency manifests found (no lockfiles, no package files): skip Phase 3. Tell the user: "No dependency manifests found — running env/credential checks only."
- Read-only filesystem: warn upfront that CP5/CP6 hook installation will fail.

---
```

- [ ] **Step 2: Add DOT flow diagram immediately after**

After the block you just inserted, add:

```
## Flow

~~~dot
digraph save_yourself {
  "Phase 0: Repo visibility"    [shape=box];
  "Phase 1: Stack detection"    [shape=box];
  "Phase 2: .env protection"    [shape=box];
  "CP2: Credential scanner"     [shape=box];
  "Pre-Phase 3: Tool check"     [shape=box];
  "Tools missing?"              [shape=diamond];
  "Offer install"               [shape=box];
  "Phase 3: Dispatch agents"    [shape=box];
  "Phase 4: Merge + report"     [shape=box];
  "CP3-CP6: Opt-in steps"       [shape=box];

  "Phase 0: Repo visibility"  -> "Phase 1: Stack detection";
  "Phase 1: Stack detection"  -> "Phase 2: .env protection";
  "Phase 2: .env protection"  -> "CP2: Credential scanner";
  "CP2: Credential scanner"   -> "Pre-Phase 3: Tool check";
  "Pre-Phase 3: Tool check"   -> "Tools missing?";
  "Tools missing?"            -> "Offer install"            [label="yes"];
  "Tools missing?"            -> "Phase 3: Dispatch agents" [label="no / resolved"];
  "Offer install"             -> "Phase 3: Dispatch agents";
  "Phase 3: Dispatch agents"  -> "Phase 4: Merge + report";
  "Phase 4: Merge + report"   -> "CP3-CP6: Opt-in steps";
}
~~~

---
```

(Use backtick fences ` ```dot ` in the actual file, not `~~~`)

- [ ] **Step 3: Verify placement**

Read `skills/save-yourself/SKILL.md` lines 1–60 and confirm:
- "When NOT to use this skill" appears before `## Phase 0`
- DOT diagram block appears between "When NOT to use" and `## Phase 0`
- `## Phase 0: Repo Visibility` header is still intact

- [ ] **Step 4: Commit**

```bash
git add skills/save-yourself/SKILL.md
git commit -m "feat: add when-not-to-use guard and flow diagram to SKILL.md"
```

---

### Task 2: Add Pre-Phase 3 tool availability check to SKILL.md

**Files:**
- Modify: `skills/save-yourself/SKILL.md`

This new section runs in the main session before any agent is dispatched. It is interactive — the user can approve tool installs here. It must appear between CP2 and Phase 3.

- [ ] **Step 1: Insert Pre-Phase 3 section before the Phase 3 header**

Find the line:
```
## Phase 3: Dependency Audit
```

Insert this block immediately BEFORE that line:

```
## Pre-Phase 3: Tool Availability Check

Narrate: "Checking required audit tools..."

For each stack detected in Phase 1, verify the required tool is installed:

| Stack  | Tool         | Check command            |
|--------|--------------|--------------------------|
| Node   | npm          | `which npm`              |
| Go     | govulncheck  | `which govulncheck`      |
| Rust   | cargo-audit  | `cargo audit --version`  |
| Python | pip-audit    | `which pip-audit`        |
| Java   | osv-scanner  | `which osv-scanner`      |

Only check rows for stacks detected in Phase 1. Skip the rest.

For each **missing** tool:
- Narrate: "`<tool>` is required for <Stack> audit but is not installed."
- Offer: "Want me to install it now?"
  - macOS install commands:
    - npm: ships with Node.js — install Node.js via `brew install node`
    - govulncheck: `go install golang.org/x/vuln/cmd/govulncheck@latest`
    - cargo-audit: `cargo install cargo-audit`
    - pip-audit: `pip install pip-audit`
    - osv-scanner: `brew install osv-scanner`
  - Linux/other: direct the user to the tool's GitHub releases page
- If user says **yes**: run the install, re-run the check command to verify. Mark stack **ready**.
- If user says **no**: mark stack **skip**. Will appear as "skipped (tool not installed)" in the Phase 4 report.
- If install fails: mark stack **skip**, include the error in the Phase 4 report.

Only stacks marked **ready** are dispatched in Phase 3.
Never dispatch an agent for a stack marked **skip** — doing so wastes a context window.

---
```

- [ ] **Step 2: Verify placement**

Read the section of SKILL.md around `## Pre-Phase 3` and confirm:
- It appears after the `## CP2: Credential File Scanner` section
- It appears before `## Phase 3`
- The table has all 5 stack rows

- [ ] **Step 3: Commit**

```bash
git add skills/save-yourself/SKILL.md
git commit -m "feat: add pre-phase-3 interactive tool availability check to SKILL.md"
```

---

### Task 3: Replace Phase 3 with parallel dispatch; update Phase 4 merge logic

**Files:**
- Modify: `skills/save-yourself/SKILL.md`

- [ ] **Step 1: Replace the Phase 3 section**

Find and replace the entire Phase 3 block. The old block is:

```
## Phase 3: Dependency Audit

Narrate: "Running dependency audit..."

For each detected stack, read the corresponding reference and follow it:

| Stack   | Reference                    | Tool        |
|---------|------------------------------|-------------|
| Node.js | @references/phase3-node.md   | npm audit   |
| Go      | @references/phase3-go.md     | govulncheck |
| Rust    | @references/phase3-rust.md   | cargo audit |
| Python  | @references/phase3-python.md | pip-audit   |
| Java    | @references/phase3-java.md   | osv-scanner |

Read only the files for detected stacks. Skip the rest.
```

Replace it with:

```
## Phase 3: Dependency Audit (Parallel Agent Dispatch)

⛔ HARD GATE: Do NOT dispatch agents until ALL of the following are complete:
  - Phase 0 (PUBLIC_REPO flag is set)
  - Phase 1 (stack list and manifest paths are finalized)
  - Phase 2 (.env findings are recorded in the session)
  - CP2 (credential findings are recorded in the session)
  - Pre-Phase 3 tool check (ready-stack list is finalized)

Narrate: "Dispatching parallel dependency audit agents for: [list ready stacks]..."

For each stack in the **ready** list, construct a dispatch prompt using this template:

```
Read @references/agents/<stack>-agent.md and follow it completely.

Context:
- PUBLIC_REPO: <value from Phase 0>
- Manifests: <list of manifest paths for this stack from Phase 1>
- Output file: .claude/save-yourself-audit-<stack>.json
```

Dispatch ALL ready-stack agents in a **single message** (multiple Agent tool calls at once)
so they run concurrently. Do NOT dispatch them sequentially.

Wait for all agents to complete before proceeding to Phase 4.
```

(The inner code fence uses backticks in the actual file.)

- [ ] **Step 2: Update Phase 4 to merge agent JSON output**

Find:
```
## Phase 4: Summary Report

Read @references/summary-format.md for the report template and fill it in.
```

Replace with:
```
## Phase 4: Summary Report

Narrate: "Merging audit results..."

For each stack dispatched in Phase 3, collect its output:

1. Check if `.claude/save-yourself-audit-<stack>.json` exists.
   - **Missing**: report "⚠️ scan incomplete for <stack> — agent produced no output" in the summary.
2. Parse the JSON file:
   - `"status": "skip"` → include `skip_reason` in summary. No findings for this stack.
   - `"status": "error"` → include `error` value in summary. No findings for this stack.
   - `"status": "ok"` → collect `findings` array. Findings are already CP1-escalated by the agent — do NOT re-apply escalation.
3. Include `coverage_note` from each agent's output as a sub-note under the stack heading in the report.

After collecting all Phase 3 findings, combine with Phase 2 and CP2 findings recorded in the main session.

Cleanup temp files:
```bash
rm -f .claude/save-yourself-audit-*.json
```

Read @references/summary-format.md for the report template and fill it in.
```

(Inner bash fence uses backticks in the actual file.)

- [ ] **Step 3: Verify changes**

Read the Phase 3 and Phase 4 sections and confirm:
- Hard gate block is present before the dispatch instructions
- Dispatch prompt template is shown (with `<stack>`, `PUBLIC_REPO`, `Manifests`, `Output file`)
- "single message" / "concurrently" instruction is explicit
- Phase 4 reads JSON files, handles missing/skip/error/ok status, does NOT re-apply CP1, and runs the cleanup command

- [ ] **Step 4: Commit**

```bash
git add skills/save-yourself/SKILL.md
git commit -m "feat: phase 3 parallel agent dispatch and phase 4 JSON merge in SKILL.md"
```

---

### Task 4: Create the 5 specialist agent reference files

**Files:**
- Create: `skills/save-yourself/references/agents/node-agent.md`
- Create: `skills/save-yourself/references/agents/go-agent.md`
- Create: `skills/save-yourself/references/agents/rust-agent.md`
- Create: `skills/save-yourself/references/agents/python-agent.md`
- Create: `skills/save-yourself/references/agents/java-agent.md`

Each agent file is a self-contained prompt for a sub-agent. It delegates audit logic to the existing `phase3-<stack>.md` reference (DRY) and wraps results in the standard JSON schema. Create the `references/agents/` directory first.

- [ ] **Step 1: Create `references/agents/node-agent.md`**

Full file content:

```markdown
# Node.js Audit Agent

You are a Node.js dependency audit specialist dispatched by /save-yourself.

## Context (injected by orchestrator)

- `PUBLIC_REPO`: true | false | unknown
- `Manifests`: list of package.json paths detected by Phase 1
- `Output file`: path to write JSON results (e.g. `.claude/save-yourself-audit-node.json`)

## Instructions

Read @references/phase3-node.md and perform the Node.js dependency audit using the
manifest paths from `Manifests`.

After completing the audit, write results to the path in `Output file` using this schema:

```json
{
  "stack": "node",
  "status": "ok",
  "findings": [
    {
      "package": "lodash",
      "installed": "4.17.15",
      "fixed": "4.17.21",
      "severity": "HIGH",
      "cve": "CVE-2021-23337",
      "description": "Prototype Pollution in lodash"
    }
  ],
  "skip_reason": null,
  "coverage_note": "npm audit scanned package-lock.json",
  "error": null
}
```

**Status rules:**
- `ok`: audit ran and produced valid output (even if `findings` is empty)
- `skip`: no `package-lock.json` found; set `skip_reason` to `"No package-lock.json found. Run npm install first."`
- `error`: npm audit returned invalid JSON or an error key; set `error` to the raw message, `findings` to `[]`

**CP1 escalation:** Apply ONLY if `PUBLIC_REPO` is `true`:
LOW → MEDIUM · MEDIUM → HIGH · HIGH → CRITICAL · CRITICAL → CRITICAL

Do NOT narrate progress. Write only to the output file.
```

- [ ] **Step 2: Create `references/agents/go-agent.md`**

Full file content:

```markdown
# Go Audit Agent

You are a Go dependency audit specialist dispatched by /save-yourself.

## Context (injected by orchestrator)

- `PUBLIC_REPO`: true | false | unknown
- `Manifests`: list of go.mod paths detected by Phase 1
- `Output file`: path to write JSON results (e.g. `.claude/save-yourself-audit-go.json`)

## Instructions

Read @references/phase3-go.md and perform the Go dependency audit.

After completing the audit, write results to the path in `Output file` using this schema:

```json
{
  "stack": "go",
  "status": "ok",
  "findings": [
    {
      "package": "golang.org/x/net",
      "installed": "0.0.0-20210226172049-4d9cf89e7dd2",
      "fixed": "0.0.0-20220225172249-27dd8689420f",
      "severity": "HIGH",
      "cve": "CVE-2022-27664",
      "description": "GO-2022-0969 — HTTP/2 server DoS via excessive HEADERS frames"
    }
  ],
  "skip_reason": null,
  "coverage_note": "govulncheck scanned ./...",
  "error": null
}
```

**Status rules:**
- `ok`: govulncheck ran and produced valid output (even if `findings` is empty)
- `skip`: govulncheck not installed; set `skip_reason` to `"govulncheck not installed. Install: go install golang.org/x/vuln/cmd/govulncheck@latest"`
- `error`: govulncheck produced a JSON `error` field or unexpected output; set `error` to the message, `findings` to `[]`

**CP1 escalation:** Apply ONLY if `PUBLIC_REPO` is `true`:
LOW → MEDIUM · MEDIUM → HIGH · HIGH → CRITICAL · CRITICAL → CRITICAL

Do NOT narrate progress. Write only to the output file.
```

- [ ] **Step 3: Create `references/agents/rust-agent.md`**

Full file content:

```markdown
# Rust Audit Agent

You are a Rust dependency audit specialist dispatched by /save-yourself.

## Context (injected by orchestrator)

- `PUBLIC_REPO`: true | false | unknown
- `Manifests`: list of Cargo.toml paths detected by Phase 1
- `Output file`: path to write JSON results (e.g. `.claude/save-yourself-audit-rust.json`)

## Instructions

Read @references/phase3-rust.md and perform the Rust dependency audit.

After completing the audit, write results to the path in `Output file` using this schema:

```json
{
  "stack": "rust",
  "status": "ok",
  "findings": [
    {
      "package": "time",
      "installed": "0.1.43",
      "fixed": "0.2.23",
      "severity": "HIGH",
      "cve": "CVE-2020-26235",
      "description": "RUSTSEC-2020-0071 — Potential segfault in time"
    }
  ],
  "skip_reason": null,
  "coverage_note": "cargo audit scanned Cargo.lock",
  "error": null
}
```

**Status rules:**
- `ok`: cargo audit ran and produced valid output (even if `findings` is empty)
- `skip`: cargo-audit not installed; set `skip_reason` to `"cargo-audit not installed. Install: cargo install cargo-audit"`
- `error`: cargo audit produced non-JSON stderr or unexpected output; set `error` to the message, `findings` to `[]`

**CP1 escalation:** Apply ONLY if `PUBLIC_REPO` is `true`:
LOW → MEDIUM · MEDIUM → HIGH · HIGH → CRITICAL · CRITICAL → CRITICAL

Do NOT narrate progress. Write only to the output file.
```

- [ ] **Step 4: Create `references/agents/python-agent.md`**

Full file content:

```markdown
# Python Audit Agent

You are a Python dependency audit specialist dispatched by /save-yourself.

## Context (injected by orchestrator)

- `PUBLIC_REPO`: true | false | unknown
- `Manifests`: list of requirements.txt / pyproject.toml / setup.py paths detected by Phase 1
- `Output file`: path to write JSON results (e.g. `.claude/save-yourself-audit-python.json`)

## Instructions

Read @references/phase3-python.md and perform the Python dependency audit.

After completing the audit, write results to the path in `Output file` using this schema:

```json
{
  "stack": "python",
  "status": "ok",
  "findings": [
    {
      "package": "Pillow",
      "installed": "8.3.1",
      "fixed": "9.0.0",
      "severity": "MEDIUM",
      "cve": "CVE-2022-22815",
      "description": "PYSEC-2022-10 — Uncontrolled resource consumption in Pillow"
    }
  ],
  "skip_reason": null,
  "coverage_note": "pip-audit scanned the active Python environment",
  "error": null
}
```

**Status rules:**
- `ok`: pip-audit ran and produced valid output (even if `findings` is empty)
- `skip`: pip-audit not installed; set `skip_reason` to `"pip-audit not installed. Install: pip install pip-audit"`
- `error`: pip-audit produced unexpected output or unrecognized JSON format; set `error` to the message, `findings` to `[]`

**CP1 escalation:** Apply ONLY if `PUBLIC_REPO` is `true`:
LOW → MEDIUM · MEDIUM → HIGH · HIGH → CRITICAL · CRITICAL → CRITICAL

Do NOT narrate progress. Write only to the output file.
```

- [ ] **Step 5: Create `references/agents/java-agent.md`**

Full file content:

```markdown
# Java Audit Agent

You are a Java dependency audit specialist dispatched by /save-yourself.

## Context (injected by orchestrator)

- `PUBLIC_REPO`: true | false | unknown
- `Manifests`: list of pom.xml / build.gradle / build.gradle.kts paths detected by Phase 1
- `Output file`: path to write JSON results (e.g. `.claude/save-yourself-audit-java.json`)

## Instructions

Read @references/phase3-java.md and perform the Java dependency audit using the
two-step CycloneDX SBOM approach described there.

After completing the audit, write results to the path in `Output file` using this schema:

```json
{
  "stack": "java",
  "status": "ok",
  "findings": [
    {
      "package": "com.fasterxml.jackson.core:jackson-databind",
      "installed": "2.12.3",
      "fixed": "2.12.7.1",
      "severity": "CRITICAL",
      "cve": "CVE-2022-42003",
      "description": "Uncontrolled resource consumption in jackson-databind"
    }
  ],
  "skip_reason": null,
  "coverage_note": "Transitive coverage: full (CycloneDX SBOM generated via Maven)",
  "error": null
}
```

**Status rules:**
- `ok`: osv-scanner ran and produced valid JSON output (even if `findings` is empty)
- `skip`: osv-scanner not installed; set `skip_reason` to `"osv-scanner not installed. Install: brew install osv-scanner (macOS) or see https://github.com/google/osv-scanner/releases"`
- `error`: osv-scanner produced empty or non-JSON output; set `error` to the raw output, `findings` to `[]`

**coverage_note values:**
- SBOM succeeded: `"Transitive coverage: full (CycloneDX SBOM generated via Maven)"` or `"...via Gradle"`
- SBOM failed, direct scan used: `"Transitive coverage: limited (SBOM generation failed — direct pom.xml scan only)"`

**CP1 escalation:** Apply ONLY if `PUBLIC_REPO` is `true`:
LOW → MEDIUM · MEDIUM → HIGH · HIGH → CRITICAL · CRITICAL → CRITICAL

Do NOT narrate progress. Write only to the output file.
```

- [ ] **Step 6: Verify all 5 files exist**

```bash
ls skills/save-yourself/references/agents/
```

Expected output:
```
go-agent.md  java-agent.md  node-agent.md  python-agent.md  rust-agent.md
```

- [ ] **Step 7: Commit**

```bash
git add skills/save-yourself/references/agents/
git commit -m "feat: add per-stack specialist agent reference files"
```

---

### Task 5: Update phase3-java.md with CycloneDX two-step approach

**Files:**
- Modify: `skills/save-yourself/references/phase3-java.md`

The current file uses `osv-scanner --format json .` (direct scan). Replace the Invocation section with the three-step CycloneDX approach and update the Known limitation note.

- [ ] **Step 1: Replace the Invocation section**

Find in `references/phase3-java.md`:
```
## Invocation

```bash
osv-scanner --format json . 2>/dev/null
```

If `command not found`:
"osv-scanner not installed.
  macOS:  `brew install osv-scanner`
  Linux/other: https://github.com/google/osv-scanner/releases"
Skip this stack.

Capture stdout. If stdout is valid JSON, parse and report findings regardless of exit code
(osv-scanner exits 1 when vulnerabilities are found — this is expected).
Only skip if stdout is empty or not valid JSON:
Report: "osv-scanner returned unexpected output — run manually: `osv-scanner .`"
Skip this stack (continue to next).
```

Replace with:

```
## Invocation

### Step 1 — Attempt CycloneDX SBOM generation (full transitive coverage)

Try Maven first (invocable without modifying pom.xml):
```bash
mvn org.cyclonedx:cyclonedx-maven-plugin:2.7.9:makeAggregateBom -q 2>/dev/null
```
If `target/bom.json` exists and is non-empty → set `SBOM_FILE=target/bom.json`.

If Maven failed or `target/bom.json` is missing/empty, try Gradle
(requires CycloneDX Gradle plugin configured in the project):
```bash
gradle cyclonedxBom -q 2>/dev/null
```
If `build/reports/bom.json` exists and is non-empty → set `SBOM_FILE=build/reports/bom.json`.

If both fail → set `SBOM_FILE=none`.

### Step 2 — Run osv-scanner

If `osv-scanner` is not installed:
"osv-scanner not installed.
  macOS:  `brew install osv-scanner`
  Linux/other: https://github.com/google/osv-scanner/releases"
Skip this stack.

With SBOM (`SBOM_FILE` is not `none`):
```bash
osv-scanner --format json --sbom $SBOM_FILE 2>/dev/null
```

Without SBOM (direct scan):
```bash
osv-scanner --format json . 2>/dev/null
```

Capture stdout. If stdout is valid JSON, parse and report findings regardless of exit code
(osv-scanner exits 1 when vulnerabilities are found — this is expected).
Only skip if stdout is empty or not valid JSON:
Report: "osv-scanner returned unexpected output — run manually: `osv-scanner .`"
Skip this stack (continue to next).

### Step 3 — Cleanup
```bash
rm -f target/bom.json target/bom.xml build/reports/bom.json
```

### Coverage note

Always include after findings in the output:
- SBOM generated: "Transitive coverage: full (CycloneDX SBOM generated via Maven/Gradle)"
- No SBOM: "Transitive coverage: limited (SBOM generation failed — direct pom.xml scan only)"
```

- [ ] **Step 2: Replace the Known limitation section**

Find:
```
## Known limitation — always show

"Note: Java scan reads pom.xml/build.gradle manifests directly.
Transitive dependency coverage is limited without a lock file.
For full transitive coverage: run `mvn dependency:tree` or `gradle dependencies`."
```

Replace with:
```
## Known limitation

If SBOM generation fails (Maven/Gradle not installed, or project doesn't build cleanly),
coverage falls back to direct pom.xml/build.gradle scan with limited transitive depth.
Always show the coverage note so the user knows which mode ran.

For projects where SBOM generation fails in CI: consider adding the CycloneDX Maven plugin
to `pom.xml` (`org.cyclonedx:cyclonedx-maven-plugin`) so `mvn cyclonedxBom` works reliably.
```

- [ ] **Step 3: Verify the updated file**

Read `references/phase3-java.md` and confirm:
- Three-step invocation is present (SBOM generation → osv-scanner → cleanup)
- `--sbom` flag is used when `SBOM_FILE` is set
- Fallback to direct scan is described for `SBOM_FILE=none`
- Coverage note is documented
- CI section at the bottom is still intact and unchanged

- [ ] **Step 4: Commit**

```bash
git add skills/save-yourself/references/phase3-java.md
git commit -m "feat: Java audit uses CycloneDX SBOM for full transitive coverage"
```

---

### Task 6: Version bump to 0.3.0 across all version files

**Files:**
- Modify: `VERSION`
- Modify: `.claude-plugin/plugin.json`
- Modify: `.claude-plugin/marketplace.json`
- Modify: `skills/save-yourself/SKILL.md` (frontmatter)
- Modify: `CHANGELOG.md`

- [ ] **Step 1: Update `VERSION`**

Change the file content from:
```
0.2.0.0
```
to:
```
0.3.0.0
```

- [ ] **Step 2: Update `plugin.json`**

Change `"version": "0.2.0"` to `"version": "0.3.0"`.

Also update the `description` field to mention parallel audits and CycloneDX:
```json
"description": "One-command security bootstrapper. Hardens .env handling, scans for credential leaks and hardcoded secrets, audits dependencies in parallel (Node, Go, Rust, Python, Java via CycloneDX SBOM), installs a gitleaks pre-commit hook (Guardian Mode), wires up Claude Code real-time secret scanning, and optionally generates a GitHub Actions CI workflow and CLAUDE.md security status block."
```

- [ ] **Step 3: Update `marketplace.json`**

Change `"version": "0.2.0"` to `"version": "0.3.0"` in the plugins array.

Also update the plugin `description` to match the new plugin.json description.

- [ ] **Step 4: Update SKILL.md frontmatter**

In `skills/save-yourself/SKILL.md`, change:
```yaml
metadata:
  version: "0.2.0"
```
to:
```yaml
metadata:
  version: "0.3.0"
```

- [ ] **Step 5: Add CHANGELOG entry**

In `CHANGELOG.md`, insert at the top (before the `## [0.2.0.0]` line):

```markdown
## [0.3.0.0] - 2026-04-26

### Added
- `references/agents/` — five specialist agent prompt templates (node, go, rust, python, java); each reads the existing phase3 reference and writes normalized JSON findings to `.claude/save-yourself-audit-<stack>.json`
- Pre-Phase 3 tool availability check: interactive install offers for missing tools before any agent is dispatched; stacks with unavailable tools are skipped and noted in the final report rather than silently failing inside agents

### Changed
- Phase 3 refactored to parallel agent dispatch: one Agent per detected stack runs concurrently, reducing wall-clock time on multi-stack repos
- Phase 4 updated to merge agent JSON output files, read `coverage_note` per stack, and clean up temp files after report generation
- SKILL.md: added "When NOT to use" guard, DOT flow diagram, and hard gate before Phase 3 dispatch
- `references/phase3-java.md`: replaced direct `osv-scanner` scan with two-step CycloneDX SBOM approach (`mvn org.cyclonedx:cyclonedx-maven-plugin:2.7.9:makeAggregateBom` → `osv-scanner --sbom`); falls back to direct scan if SBOM generation fails; always reports transitive coverage level in output

### Fixed
- Java audit no longer silently loses transitive dependencies when Maven is available in the environment
- Missing audit tools now surface interactively before scan begins, not as silent skips inside agent output
```

- [ ] **Step 6: Verify JSON files are valid**

```bash
python3 -c "import json; json.load(open('.claude-plugin/plugin.json'))" && echo "plugin.json OK"
python3 -c "import json; json.load(open('.claude-plugin/marketplace.json'))" && echo "marketplace.json OK"
```

Expected:
```
plugin.json OK
marketplace.json OK
```

- [ ] **Step 7: Commit**

```bash
git add VERSION .claude-plugin/plugin.json .claude-plugin/marketplace.json skills/save-yourself/SKILL.md CHANGELOG.md
git commit -m "chore: bump version to 0.3.0 across all version files"
```
