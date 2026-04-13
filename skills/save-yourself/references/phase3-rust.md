# Phase 3: Rust Dependency Audit

Narrate: "Running cargo audit..."

```bash
cargo audit --json 2>/dev/null
```

If `command not found`:
"cargo-audit not installed. Install: `cargo install cargo-audit`"

If non-JSON stderr: "cargo audit failed — run manually to investigate."

Apply CP1 escalation.

## CI section (for CP3 GitHub Actions)

```yaml
- name: Rust security audit
  if: hashFiles('Cargo.toml') != ''
  uses: rustsec/audit-check@v2
  with:
    token: ${{ secrets.GITHUB_TOKEN }}
```
