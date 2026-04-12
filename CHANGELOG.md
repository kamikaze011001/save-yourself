# Changelog

All notable changes to save-yourself are documented here.

## [0.1.0.0] - 2026-04-12

### Added
- `skills/save-yourself/SKILL.md` — full 6-phase security bootstrapper skill for Claude Code
  - Phase 0: repo visibility check with CP1 severity escalation for public repos
  - Phase 1: stack detection (Node.js, Go, Rust) with monorepo support
  - Phase 2: .env protection suite (gitignore, tracking check, history scan, secret pattern scan, .env.example generation/merge)
  - CP2: credential file scanner (GCP keys, .npmrc, Docker configs, .pem/.key files with content-based severity)
  - Phase 3: dependency audit (npm audit with v6/v7+ JSON format handling, govulncheck, cargo audit)
  - Phase 4: structured summary report with auto-fixed vs action-required sections
  - CP3: optional GitHub Actions CI workflow generator (Node, Go, Rust, .env history check)
  - CP4: optional CLAUDE.md security status block with idempotent replace
- `skills/save-yourself/references/secret-patterns.md` — regex reference, sed truncation command, full git grep invocation with macOS BSD sed compatibility notes
- `.claude-plugin/plugin.json` — plugin metadata (name, version, keywords, repository)
- `.claude-plugin/marketplace.json` — marketplace listing for `claude plugin install`
- `README.md` — install instructions, usage, supported stacks, what auto-fixes vs manual
- `TODOS.md` — P2 backlog: standalone CLI distribution, Python/Java support, Guardian Mode pre-commit hooks
- `.gitignore` — excludes `.claude/` (developer workspace config) and `.gstack/` (build tool metadata)
