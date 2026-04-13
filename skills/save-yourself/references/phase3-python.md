# Phase 3: Python Dependency Audit

Narrate: "Running pip-audit (this may take 30s on large projects)..."

Detection: any of `requirements.txt` / `pyproject.toml` / `setup.py` found in Phase 1.

## Invocation

Run pip-audit from the project root. If a virtualenv exists at `.venv/`, prefer it:

```bash
[ -f .venv/bin/pip-audit ] && .venv/bin/pip-audit --format json 2>/dev/null \
  || pip-audit --format json 2>/dev/null
```

If `command not found`:
"pip-audit not installed. Install: `pip install pip-audit`"
Skip this stack.

If pip-audit exits non-zero or output is not valid JSON:
Report: "pip-audit returned unexpected output — run manually: pip-audit"
Skip this stack (continue to next).

## Output parsing

Detect format first:
- If output is a JSON array → post-1.0 format: `[{ name, version, vulns:[{id, fix_versions, aliases}] }]`
- If output is a JSON object with `"dependencies"` key → pre-1.0 format: `{ dependencies:[{name, version, vulns:[]}] }`
- If neither → report: "pip-audit returned unrecognized format. Upgrade: `pip install --upgrade pip-audit`"

For post-1.0 format, normalize each result:
```
package:           result.name
installed_version: result.version
fixed_version:     vuln.fix_versions[0] (or "no fix available")
severity:          not provided by pip-audit — default MEDIUM unless CVE maps to NVD severity
cve:               vuln.aliases[] filtered for CVE-* prefix, first match
description:       vuln.id
```

Apply CP1 escalation to all findings.

## Known limitation

pip-audit audits the active Python environment, not just requirements.txt.
If the virtualenv is not activated, coverage may be incomplete.
Always note: "Note: pip-audit scanned the active Python environment."

## CI section (for CP3 GitHub Actions)

```yaml
- name: Python dependency audit
  if: hashFiles('requirements.txt') != '' || hashFiles('pyproject.toml') != '' || hashFiles('setup.py') != ''
  run: |
    pip install --quiet pip-audit
    pip install --quiet -r requirements.txt 2>/dev/null || pip install --quiet -e . 2>/dev/null \
      || echo "::warning::pip install failed — pip-audit may scan an incomplete environment"
    pip-audit --format json | python3 -c "
    import json,sys
    data=json.load(sys.stdin)
    vulns=[v for r in data for v in r.get('vulns',[])]
    if vulns:
      print(f'::error::{len(vulns)} vulnerable Python package(s) found')
      sys.exit(1)
    "
```
