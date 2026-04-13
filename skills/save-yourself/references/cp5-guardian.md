# CP5: Guardian Mode — gitleaks Pre-Commit Hook

## Step 1: Check gitleaks installed

```bash
gitleaks version 2>/dev/null | grep -q . && echo "INSTALLED" || echo "MISSING"
```

If MISSING:
"gitleaks not installed. Install first:
  macOS:  `brew install gitleaks`
  Linux:  https://github.com/gitleaks/gitleaks/releases
Then re-run /save-yourself to complete Guardian Mode setup."
Stop CP5.

## Step 2: Check existing pre-commit hook

```bash
[ -f .git/hooks/pre-commit ] && echo "EXISTS" || echo "MISSING"
```

If EXISTS:
```bash
grep -q 'gitleaks protect' .git/hooks/pre-commit && echo "WIRED" || echo "NEEDS_APPEND"
```

- WIRED → "gitleaks already in pre-commit hook. Skipping." Stop CP5.
- NEEDS_APPEND → append the block below to the existing file.

If MISSING → create new file with shebang + block below.

## Step 3: Content to write or append

New file (include shebang):
```sh
#!/bin/sh
# === save-yourself: gitleaks secret scanner ===
if command -v gitleaks >/dev/null 2>&1; then
  gitleaks protect --staged --redact -v
  EXIT_CODE=$?
  if [ $EXIT_CODE -eq 1 ]; then
    echo ""
    echo "SECRET DETECTED: commit blocked by gitleaks."
    echo "Remove the secret or move it to .env, then try again."
    echo "(Use 'git commit --no-verify' to bypass if this is a false positive.)"
    exit 1
  elif [ $EXIT_CODE -gt 1 ]; then
    echo "gitleaks exited with error (exit $EXIT_CODE) — commit not blocked."
    echo "Check your gitleaks installation: gitleaks version"
  fi
fi
# === end save-yourself ===
```

Append to existing file (omit shebang, include block):
```sh
# === save-yourself: gitleaks secret scanner ===
if command -v gitleaks >/dev/null 2>&1; then
  gitleaks protect --staged --redact -v
  EXIT_CODE=$?
  if [ $EXIT_CODE -eq 1 ]; then
    echo ""
    echo "SECRET DETECTED: commit blocked by gitleaks."
    echo "Remove the secret or move it to .env, then try again."
    echo "(Use 'git commit --no-verify' to bypass if this is a false positive.)"
    exit 1
  elif [ $EXIT_CODE -gt 1 ]; then
    echo "gitleaks exited with error (exit $EXIT_CODE) — commit not blocked."
    echo "Check your gitleaks installation: gitleaks version"
  fi
fi
# === end save-yourself ===
```

Make executable:
```bash
chmod +x .git/hooks/pre-commit
```

Notify: "Guardian Mode enabled. gitleaks will scan staged files before every commit."

Verify:
```bash
cat .git/hooks/pre-commit | grep -q gitleaks && echo "Guardian Mode: ACTIVE"
```

To remove: Edit `.git/hooks/pre-commit` and delete the block between:
```
# === save-yourself: gitleaks secret scanner ===
  ...
# === end save-yourself ===
```
If that was the only content: `rm .git/hooks/pre-commit`

Tell the user (as part of the Phase 4 summary or as a final status note):
```
+ Guardian Mode: gitleaks pre-commit hook installed (.git/hooks/pre-commit)
```
