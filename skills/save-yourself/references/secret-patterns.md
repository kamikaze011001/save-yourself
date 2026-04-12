# Secret Pattern Reference

Used by Phase 2d (secret pattern scan in tracked files).

## Primary scan pattern (git grep -P)

```
(?i)(api[_-]?key|secret|token|password|passwd|pwd)\s*=\s*\S+
```

## False positive exclusion filter (grep -viE)

Exclude lines containing any of:

```
(example|placeholder|changeme|your_|xxx|<|>|\$\{|\$\()
```

This catches:
- `API_KEY=your_api_key_here` — excluded by `your_`
- `PASSWORD=<your-password>` — excluded by `<`
- `TOKEN=${MY_TOKEN}` — excluded by `\$\{`
- `SECRET=$(get_secret)` — excluded by `\$\(`
- `# API_KEY=changeme` — excluded by `changeme`

## Truncation sed (macOS BSD sed compatible)

```bash
sed -E 's/((api[_-]?key|secret|token|password|passwd|pwd)[[:space:]]*=[[:space:]]*.{4})[^[:space:]]*/\1.../i'
```

Pattern-aware: matches on the key name, then captures first 4 chars of the value.
Does NOT match the first `=` in the line (avoids corrupting paths containing `=`).

## Full git grep invocation

```bash
git grep -P -n -I \
  '(?i)(api[_-]?key|secret|token|password|passwd|pwd)\s*=\s*\S+' \
  -- ':!node_modules/' ':!vendor/' ':!dist/' ':!.git/' ':!*.min.js' ':!*.min.css' \
  2>/dev/null \
  | grep -viE '(example|placeholder|changeme|your_|xxx|<|>|\$\{|\$\()' \
  | sed -E 's/((api[_-]?key|secret|token|password|passwd|pwd)[[:space:]]*=[[:space:]]*.{4})[^[:space:]]*/\1.../i'
```

Flags:
- `-P` — Perl regex (required for `\s`, `\S`, `?i` flag)
- `-n` — include line numbers in output
- `-I` — skip binary files
- `--` — separates pathspecs from pattern; prevents ARG_MAX overflow
- `:!path/` — pathspec exclusion (gitignore-style, built into git grep)

## Scope note (always include after scan output)

```
Note: Secret pattern scan covers tracked files at HEAD only.
Secrets deleted from code but still in git history are not caught here.
To scan history manually: git log -p --all -- . | grep -iE '(api_key|secret|token|password)\s*='
```
