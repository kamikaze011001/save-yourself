# Changelog

All notable changes to save-yourself are documented here.

## [0.2.0.0] - 2026-04-13

### Added
- `skills/save-yourself/references/` — conductor pattern: SKILL.md is now ~150-line conductor,
  all implementation detail lives in per-phase reference files loaded on-demand
- `references/phase3-python.md` — Python dependency audit via pip-audit with pre-1.0/post-1.0
  JSON schema format detection and .venv/ virtualenv preference
- `references/phase3-java.md` — Java dependency audit via osv-scanner with transitive coverage
  limitation documented
- `references/cp5-guardian.md` — Guardian Mode: gitleaks pre-commit hook with exit code 1 vs 2+
  distinction (secret found vs tool error), idempotent install, uninstall instructions
- `references/cp6-cc-hooks.md` — Claude Code Defense Layer: PreToolUse hook for Write/Edit/Bash
  with SAFE_CONTENT env var pattern (avoids pipe+heredoc conflict and bash injection), idempotent
  settings.json merge, uninstall instructions

### Changed
- SKILL.md refactored to conductor pattern (~155 lines, was 463)
- Phase 1 stack detection expanded to include Python and Java manifests
- CP3 GitHub Actions workflow updated with Python (pip-audit) and Java (google/osv-scanner-action@v1) steps
- `references/secret-patterns.md` now actually referenced from phase2-env.md (was orphaned in v0.1)
- Version bumped to 0.2.0 in SKILL.md frontmatter and marketplace.json

### Fixed
- CP6 hook MATCH extraction: SAFE_CONTENT env var replaces pipe+heredoc pattern
  (heredoc overrides pipe stdin; content was always ""; hook silently did nothing)
- CP6 deny path: SAFE_MATCH env var replaces triple-quoted bash injection risk
- CP5 gitleaks exit code handling: exit 1 (secret found) blocks commit; exit 2+ (tool error) warns only

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
