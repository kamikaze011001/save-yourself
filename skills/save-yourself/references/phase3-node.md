# Phase 3: Node.js Dependency Audit

Narrate: "Running npm audit..."

If `package.json` exists but no `package-lock.json`:
"Run `npm install` first to generate a lockfile, then re-run `/save-yourself`." Skip.

Monorepo: find all `package.json` files up to 2 levels deep (matches Phase 1 stack detection depth in SKILL.md — update both if changing scan depth), excluding `node_modules/`:
```bash
find . -maxdepth 2 -name "package.json" ! -path "*/node_modules/*"
```
Cap at 20. If more: "Scope limited to 20 of N packages. Run `npm audit` in remaining packages manually."

For each `<dir>` (the directory containing the `package.json`):
```bash
npm audit --prefix <dir> --json 2>/dev/null
```

Never `cd` into the directory — use `--prefix` so the shell working directory stays at the project root throughout.

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

## CI section (for CP3 GitHub Actions)

```yaml
- name: Node.js dependency audit
  if: hashFiles('package.json') != ''
  run: npm audit --audit-level=high
```
