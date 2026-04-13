# Phase 2: .env Protection Suite

## 2a. .gitignore coverage

```bash
grep -qE '^/?\.env([[:space:]]|#|$)|^\.env\*' .gitignore 2>/dev/null || echo "MISSING"
```

If missing: **auto-add without asking**. Append these lines to `.gitignore`:
```
.env
.env.*
!.env.example
```
This covers `.env`, `.env.local`, `.env.production`, `.env.staging`, `.env.development`, `.env.test`,
and all other variants — while explicitly un-ignoring `.env.example`.
Notify: "Added `.env` and `.env.*` to `.gitignore` (`.env.example` excluded)."

## 2b. Currently tracked

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

## 2c. Git history scan

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

## 2d. Secret pattern scan

Narrate: "Scanning tracked files for hardcoded secrets..."

Use the full git grep invocation from @references/secret-patterns.md.

Cap output at 500 lines. If more, report: "Scope limited — run git grep manually for full scan."

Report findings grouped by file:
```
MEDIUM: src/config.js:42 — API_KEY=prod_ab...
→ Remove from code, move to .env, recommit
```
(Apply CP1 escalation.)

Always add after scan output (scope note from @references/secret-patterns.md):
```
Note: Secret pattern scan covers tracked files at HEAD only.
Secrets deleted from code but still in history are not caught here.
To scan history: git log -p --all -- . | grep -iE '(api_key|secret|token|password)\s*='
```

## 2e. .env.example generation/merge

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
