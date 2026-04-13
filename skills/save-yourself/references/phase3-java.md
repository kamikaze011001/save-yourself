# Phase 3: Java Dependency Audit

Narrate: "Running osv-scanner for Java..."

Detection: `pom.xml` or `build.gradle` or `build.gradle.kts` found in Phase 1.

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

## Output parsing

osv-scanner JSON:
```json
{
  "results": [{
    "packages": [{
      "package": { "name": "...", "version": "...", "ecosystem": "..." },
      "vulnerabilities": [{ "id": "...", "aliases": [], "severity": [], "affected": [], "summary": "..." }]
    }]
  }]
}
```

Normalize each vuln:
```
package:           pkg.package.name
installed_version: pkg.package.version
fixed_version:     from affected[].ranges[].events[] where type=fixed, first match
severity:          vulnerability.severity[].score mapped to LOW/MEDIUM/HIGH/CRITICAL
                   fallback: MEDIUM if not provided
cve:               vulnerability.aliases[] filtered for CVE-* prefix
description:       vulnerability.id + " — " + vulnerability.summary (if present)
```

Apply CP1 escalation to all findings.

## Known limitation — always show

"Note: Java scan reads pom.xml/build.gradle manifests directly.
Transitive dependency coverage is limited without a lock file.
For full transitive coverage: run `mvn dependency:tree` or `gradle dependencies`."

## CI section (for CP3 GitHub Actions)

```yaml
- name: Java dependency scan (osv-scanner)
  if: hashFiles('pom.xml') != '' || hashFiles('build.gradle') != '' || hashFiles('build.gradle.kts') != ''
  uses: google/osv-scanner-action@v1
  with:
    scan-args: |-
      --format=json
      ./
```
