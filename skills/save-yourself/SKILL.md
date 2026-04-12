---
name: save-yourself
description: >
  One-command security bootstrapper. Trigger when the user runs /save-yourself,
  wants to harden their project security, check for leaked credentials, scan for
  hardcoded secrets, audit dependencies for vulnerabilities, or set up .env
  protection. Also trigger when a user says "secure my project", "check for
  security issues", or "am I leaking credentials".
metadata:
  version: "0.1.0"
  author: "kamikaze011001"
---

# /save-yourself — Security Bootstrapper

One command. Leave the project hardened.

Runs in phases: detect stack, harden .env handling, scan for credential files and
hardcoded secrets, audit dependencies, then offer GitHub Actions CI and a CLAUDE.md
security status block.

Before starting, tell the user: "Running /save-yourself security scan. This will
check your .env setup, scan for leaked credentials, and audit dependencies. I'll
narrate each step as I go."

---

## Phase 0: Repo Visibility

Narrate: "Checking repo visibility..."

```bash
IS_PUBLIC=$(gh repo view --json isPrivate -q '.isPrivate' 2>/dev/null)
```

- Output `false` → repo is public. Set `PUBLIC_REPO=true`.
  Show: `⚠️  PUBLIC REPO: credential leaks are indexed immediately by search engines and bots.`
- Output `true` → repo is private. Set `PUBLIC_REPO=false`.
- Any error or empty → set `PUBLIC_REPO=unknown`.
  Note: "Could not verify repo visibility — assuming private. Severity escalation will not apply."

**CP1 rule** (active for the rest of the skill when `PUBLIC_REPO=true`):
All severity findings get +1 tier: LOW→MEDIUM, MEDIUM→HIGH, HIGH→CRITICAL.
CRITICAL stays CRITICAL. Apply before reporting any finding.

---

## Phase 1: Stack Detection

Narrate: "Detecting project stack..."

```bash
[ -f package.json ] && echo "NODE"
[ -f go.mod ]       && echo "GO"
[ -f Cargo.toml ]   && echo "RUST"
```

Also check one level down for monorepos:

```bash
find . -maxdepth 2 -name "package.json" ! -path "*/node_modules/*"
find . -maxdepth 2 -name "go.mod"
find . -maxdepth 2 -name "Cargo.toml"
```

Report: "Found [detected stacks] in this project."

If no lockfile found at root or one level down: ask the user which stack they're using.
Accept "node", "go", "rust" and proceed with that audit only.

---

## Phase 2: .env Protection Suite

Narrate: "Checking .env protection..."

### 2a. .gitignore coverage

```bash
grep -qE '^\.env$' .gitignore 2>/dev/null || echo "MISSING"
```

If missing: **auto-add without asking**. Append `.env`, `.env.local`, `.env.*.local` to `.gitignore`.
Notify: "Added `.env` (and variants) to `.gitignore`."

### 2b. Currently tracked

```bash
git ls-files --error-unmatch .env 2>/dev/null && echo "TRACKED"
```

If `TRACKED`:
```
CRITICAL: .env is currently tracked by git.
.gitignore alone does NOT untrack a file already in the index.
→ Run: git rm --cached .env && git commit -m "chore: untrack .env"
```
(Apply CP1 escalation.)

### 2c. Git history scan

Narrate: "Scanning git history for .env commits..."

```bash
git log --all --full-history -- \
  '.env' '.env.local' '.env.production' '.env.staging' '.env.development' '.env.test' \
  2>/dev/null | grep -q . && echo "FOUND_IN_HISTORY" || echo "CLEAN"
```

If `git log` fails (empty repo): output "No git history found — skipping history checks." Continue.

If `FOUND_IN_HISTORY`:
```
CRITICAL: .env found in git history.
Even if the file was deleted, it is still readable in history.
→ Run: git filter-repo --path .env --invert-paths
→ Then force-push and notify all collaborators to re-clone.
→ Rotate any credentials that were in the file.
```

### 2d. Secret pattern scan

Narrate: "Scanning tracked files for hardcoded secrets..."

```bash
git grep -P -n -I \
  '(?i)(api[_-]?key|secret|token|password|passwd|pwd)\s*=\s*\S+' \
  -- ':!node_modules/' ':!vendor/' ':!dist/' ':!.git/' ':!*.min.js' ':!*.min.css' \
  2>/dev/null \
  | grep -viE '(example|placeholder|changeme|your_|xxx|<|>|\$\{|\$\()' \
  | sed -E 's/((api[_-]?key|secret|token|password|passwd|pwd)[[:space:]]*=[[:space:]]*.{4})[^[:space:]]*/\1.../i'
```

Cap output at 500 lines. If more, report: "Scope limited — run git grep manually for full scan."

Report findings grouped by file:
```
MEDIUM: src/config.js:42 — API_KEY=prod_ab...
→ Remove from code, move to .env, recommit
```
(Apply CP1 escalation.)

Always add after scan output:
```
Note: Secret pattern scan covers tracked files at HEAD only.
Secrets deleted from code but still in history are not caught here.
To scan history: git log -p --all -- . | grep -iE '(api_key|secret|token|password)\s*='
```

### 2e. .env.example generation/merge

Skip if `.env` does not exist.

Key parser function:
```bash
parse_env_keys() {
  grep -E '^\s*(export\s+)?[A-Za-z_][A-Za-z0-9_]*=' "$1" \
  | sed -E 's/^\s*(export\s+)?([A-Za-z_][A-Za-z0-9_]*)=.*/\2/'
}
```

If `.env.example` does not exist: **auto-create** with all keys, values stripped.
Format: `KEY=` (trailing `=` retained). Notify: "Created `.env.example` (N keys)."

If `.env.example` exists: merge new keys only:
```bash
ENV_KEYS=$(parse_env_keys .env)
EXAMPLE_KEYS=$(parse_env_keys .env.example)
NEW_KEYS=$(comm -23 <(echo "$ENV_KEYS" | sort) <(echo "$EXAMPLE_KEYS" | sort))
for KEY in $NEW_KEYS; do echo "${KEY}=" >> .env.example; done
```
Notify: "Merged N new keys into existing `.env.example`."

---

## CP2: Credential File Scanner

Narrate: "Scanning for tracked credential files..."

### Step 1: Presence check

```bash
git ls-files | grep -iE \
  '(serviceaccountkey|credentials|keyfile|docker-config|\.npmrc|\.p12|\.pfx|\.pem|\.key)$'
git ls-files | grep -F '.docker/config.json'
```

### Step 2: Content checks per file type

**serviceAccountKey.json, credentials.json, keyfile.json:**
Flag CRITICAL immediately — GCP service account files.

**`.npmrc`:**
```bash
git show HEAD:.npmrc 2>/dev/null | grep -qi '_authToken='
```
- `_authToken=` found: HIGH
- Exists, no token: LOW (informational)

**`docker-config.json` or `.docker/config.json`:**
```bash
git show HEAD:docker-config.json 2>/dev/null \
  | python3 -c "import sys,json; d=json.load(sys.stdin); print('HAS_AUTHS') if d.get('auths') else None" \
  2>/dev/null
# python3 not available fallback:
git show HEAD:docker-config.json 2>/dev/null | grep -q '"auths"' && echo "HAS_AUTHS"
```
- `HAS_AUTHS`: HIGH
- File exists, no `auths` key: MEDIUM

**`.pem`, `.key` files:**
```bash
git show HEAD:<file> 2>/dev/null | grep -q 'PRIVATE KEY' && echo CRITICAL || echo MEDIUM
```
- Contains `PRIVATE KEY` (RSA, EC, etc.): CRITICAL
- Contains `BEGIN CERTIFICATE` only (public cert): MEDIUM

**`.p12`, `.pfx` files:** HIGH (certificate bundles typically contain private keys)

### Output format

```
[SEVERITY] <filename> — <one-line reason>
→ git rm --cached <filename>
→ Add to .gitignore
→ Rotate any credentials this file contained
```

Apply CP1 escalation for public repos. No auto-fix — manual review required.

---

## Phase 3: Dependency Audit

### Node.js

Narrate: "Running npm audit..."

If `package.json` exists but no `package-lock.json`:
"Run `npm install` first to generate a lockfile, then re-run `/save-yourself`." Skip.

Monorepo: find all `package.json` files up to 2 levels deep, excluding `node_modules/`:
```bash
find . -maxdepth 2 -name "package.json" ! -path "*/node_modules/*"
```
Cap at 20. If more: "Scope limited to 20 of N packages. Run `npm audit` in remaining packages manually."

For each:
```bash
npm audit --json 2>/dev/null
```

Parse — detect format first:
```
if report.auditReportVersion === 2:
  # npm v7+: use report.vulnerabilities
  for each pkg in vulnerabilities:
    extract: package, range (installed), severity, via[].title, via[].cve
else:
  # npm v6: use report.advisories
  for each id in advisories:
    extract: module_name, findings[0].version, severity, title, cves[0]
```

Normalize to: `{ package, installed_version, fixed_version, severity, cve, description }`

If JSON has `error` key or fails to parse: "npm audit failed: [error]. Run manually."

Apply CP1 escalation to all findings.

### Go

Narrate: "Running govulncheck (this may take 30s)..."

```bash
govulncheck -json ./... 2>/dev/null
```

If `command not found`:
"govulncheck not installed. Install: `go install golang.org/x/vuln/cmd/govulncheck@latest`
Note: install the latest version — older versions may silently miss vulnerabilities."

If JSON `error` field present: report as-is.

Apply CP1 escalation.

### Rust

Narrate: "Running cargo audit..."

```bash
cargo audit --json 2>/dev/null
```

If `command not found`:
"cargo-audit not installed. Install: `cargo install cargo-audit`"

If non-JSON stderr: "cargo audit failed — run manually to investigate."

Apply CP1 escalation.

---

## Phase 4: Summary Report

Explain each finding in plain English first, then output the structured report:

```
== SAVE YOURSELF REPORT ==
Scanned: {ISO date}
Repo: {public / private / unknown}

[FIXED AUTOMATICALLY]
  + Added .env to .gitignore
  + Created .env.example (7 keys)

[ACTION REQUIRED]
  CRITICAL: .env found in git history (branch: main, commit: abc1234)
    → git filter-repo --path .env --invert-paths
    → Force-push, notify collaborators to re-clone, rotate credentials

  HIGH: 2 vulnerable dependencies in package.json
    - lodash@4.17.11 → upgrade to 4.17.21 (CVE-2021-23337: command injection)
    - axios@0.21.0  → upgrade to 0.21.2  (CVE-2021-3749: ReDoS)
    → npm install lodash@4.17.21 axios@0.21.2

  MEDIUM: 1 potential secret in tracked files
    - src/config.js:42 — API_KEY=prod_ab...
    → Remove from code, move to .env, recommit

[PASSED]
  + .env not in git history
  + .env.example up to date
  + Go: no vulnerabilities found

[SKIPPED — install tool to enable]
  Rust: cargo-audit not installed
    → cargo install cargo-audit — then re-run /save-yourself
```

Always end the report with:
```
Note: This scan covers common patterns only. It is not a substitute for a professional security audit.
```

---

## CP3: GitHub Actions Workflow (opt-in)

After Phase 4, offer:
> "Want me to create a GitHub Actions workflow that runs these checks on every PR?"

If no supported lockfile found: "No supported language detected — workflow would run zero checks. Skipping."

If `.github/workflows/save-yourself.yml` already exists: "A workflow already exists. Overwrite it?"

If user says yes (or no existing file), create `.github/workflows/save-yourself.yml`:

```yaml
name: Security Audit
permissions:
  contents: read
on:
  pull_request:
    branches: [main, master]
  push:
    branches: [main, master]

jobs:
  security:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0

      - name: Node.js dependency audit
        if: hashFiles('package.json') != ''
        run: npm audit --audit-level=high

      - name: Go vulnerability check
        if: hashFiles('go.mod') != ''
        run: |
          go install golang.org/x/vuln/cmd/govulncheck@latest
          govulncheck ./...

      - name: Rust security audit
        if: hashFiles('Cargo.toml') != ''
        uses: rustsec/audit-check@v2
        with:
          token: ${{ secrets.GITHUB_TOKEN }}

      - name: Check for .env in git history
        run: |
          if git log --all --full-history -- .env .env.local .env.production .env.staging | grep -q .; then
            echo "::error::.env found in git history — run git filter-repo to remove"
            exit 1
          fi
```

Generate the file. Do not auto-commit. Tell the user to review and commit it.

---

## CP4: CLAUDE.md Security Status (opt-in)

After CP3 resolves, offer:
> "Want me to add a security status block to CLAUDE.md so future sessions inherit this context?"

If yes:

Check if `CLAUDE.md` exists. Create it if missing.

Check for existing section:
```bash
grep -n "^## Security Status" CLAUDE.md 2>/dev/null
```

If section exists: replace it entirely using the Edit tool.
- `old_string`: the full `## Security Status` block from `## Security Status\n` through the next `## ` heading or EOF.
- `new_string`: the updated block.

If section does not exist: append to EOF.

Block to write:
```markdown
## Security Status
Last scanned: {ISO date}
By: /save-yourself skill
Summary: {N} issues found, {M} auto-fixed, {K} action required
Critical: {list of unresolved CRITICAL findings, or "none"}
→ Re-run /save-yourself before each release.
```

N = total findings. M = auto-applied fixes (.gitignore entries + .env.example). K = ACTION REQUIRED items.

---

## Failure modes

| Situation | Behavior |
|---|---|
| Empty repo (no commits) | Phase 2c: "No git history — skipping history checks." Continue. |
| No lockfile found | Phase 1: ask user which stack. Phase 3: skip undetected stacks. |
| `package.json` but no lockfile | Phase 3: "Run `npm install` first." Skip npm audit. |
| `govulncheck` not installed | Phase 3: skip with install instructions. |
| `cargo audit` not installed | Phase 3: skip with install instructions. |
| `gh` not installed or unauthenticated | Phase 0: `PUBLIC_REPO=unknown`, no CP1 escalation. |
| `python3` not available (docker check) | CP2: fall back to `grep '"auths"'`, flag MEDIUM. |
| Monorepo > 20 packages | Phase 3: scan first 20, report scope limit. |
| `save-yourself.yml` workflow exists | CP3: ask before overwriting. |
| `## Security Status` exists in CLAUDE.md | CP4: replace in place (idempotent). |

## What this skill does NOT do

- `npm audit fix` — can introduce breaking changes, manual review required
- `git filter-repo` — destructive, user must run manually
- Edit source files to remove hardcoded secrets — manual
- Python (`pip-audit`), Java (OWASP) — v2
- Pre-commit hooks (Guardian Mode) — v2
- Full git history scan for non-.env files — too slow; manual command shown above
