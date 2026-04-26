# Phase 3: Java Dependency Audit

Narrate: "Running osv-scanner for Java..."

Detection: `pom.xml` or `build.gradle` or `build.gradle.kts` found in Phase 1.

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

## Known limitation

If SBOM generation fails (Maven/Gradle not installed, or project doesn't build cleanly),
coverage falls back to direct pom.xml/build.gradle scan with limited transitive depth.
Always show the coverage note so the user knows which mode ran.

For projects where SBOM generation fails in CI: consider adding the CycloneDX Maven plugin
to `pom.xml` (`org.cyclonedx:cyclonedx-maven-plugin`) so `mvn cyclonedxBom` works reliably.

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
