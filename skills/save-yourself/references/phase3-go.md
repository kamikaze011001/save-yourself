# Phase 3: Go Dependency Audit

Narrate: "Running govulncheck (this may take 30s)..."

```bash
govulncheck -json ./... 2>/dev/null
```

If `command not found`:
"govulncheck not installed. Install: `go install golang.org/x/vuln/cmd/govulncheck@latest`
Note: install the latest version — older versions may silently miss vulnerabilities."

If JSON `error` field present: report as-is.

Apply CP1 escalation.

## CI section (for CP3 GitHub Actions)

```yaml
- name: Go vulnerability check
  if: hashFiles('go.mod') != ''
  run: |
    go install golang.org/x/vuln/cmd/govulncheck@latest
    govulncheck ./...
```
