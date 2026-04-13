# Phase 4: Summary Report

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
