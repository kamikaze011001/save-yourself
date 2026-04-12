# save-yourself

One-command security bootstrapper for Claude Code projects.

Run it once at project start. Leave the project hardened.

## What it does

1. **Repo visibility check** — detects public repos and escalates severity for all findings
2. **Stack detection** — Node.js, Go, Rust (checks root + monorepo subpackages)
3. **.env protection suite:**
   - Adds `.env` to `.gitignore` automatically
   - Warns if `.env` is currently tracked by git
   - Scans git history for past `.env` commits
   - Scans tracked files for hardcoded secrets (API keys, tokens, passwords)
   - Creates or merges `.env.example`
4. **Credential file scanner** — flags tracked `.npmrc`, GCP service account keys, Docker configs, `.pem`/`.key` files
5. **Dependency audit** — `npm audit`, `govulncheck`, `cargo audit` with unified output
6. **GitHub Actions workflow** (opt-in) — CI that runs all checks on every PR
7. **CLAUDE.md security status** (opt-in) — so future sessions inherit the security context

## Install

```bash
claude plugin install save-yourself
```

## Usage

```
/save-yourself
```

Run before each release.

## Supported stacks (v1)

- Node.js / TypeScript (`npm audit`)
- Go (`govulncheck`)
- Rust (`cargo audit`)

Python and Java are deferred to v2.

## What it fixes automatically

- Adds `.env` and variants to `.gitignore`
- Creates or merges `.env.example`

Everything else (dep upgrades, secret removal, git history rewriting) requires your
explicit action — with exact commands provided.

## License

MIT
