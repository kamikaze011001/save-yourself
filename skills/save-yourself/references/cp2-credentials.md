# CP2: Credential File Scanner

## Step 1: Presence check

```bash
git ls-files | grep -iE \
  '(serviceaccountkey|credentials|keyfile|docker-config|\.npmrc|\.p12|\.pfx|\.pem|\.key)$'
git ls-files | grep -F '.docker/config.json'
```

## Step 2: Content checks per file type

**serviceAccountKey.json, credentials.json, keyfile.json:**
Flag CRITICAL immediately — GCP service account files.

**`.npmrc`:**
```bash
git show HEAD:.npmrc 2>/dev/null | grep -qi '_authToken='
```
- `_authToken=` found: HIGH
- Exists, no token: LOW (informational)

**`docker-config.json` or `.docker/config.json`:**
```bash
git show HEAD:docker-config.json 2>/dev/null \
  | python3 -c "import sys,json; d=json.load(sys.stdin); print('HAS_AUTHS') if d.get('auths') else None" \
  2>/dev/null
# python3 not available fallback:
git show HEAD:docker-config.json 2>/dev/null | grep -q '"auths"' && echo "HAS_AUTHS"
```
- `HAS_AUTHS`: HIGH
- File exists, no `auths` key: MEDIUM

**`.pem`, `.key` files:**
```bash
git show HEAD:<file> 2>/dev/null | grep -q 'PRIVATE KEY' && echo CRITICAL || echo MEDIUM
```
- Contains `PRIVATE KEY` (RSA, EC, etc.): CRITICAL
- Contains `BEGIN CERTIFICATE` only (public cert): MEDIUM

**`.p12`, `.pfx` files:** HIGH (certificate bundles typically contain private keys)

## Output format

```
[SEVERITY] <filename> — <one-line reason>
→ git rm --cached <filename>
→ Add to .gitignore
→ Rotate any credentials this file contained
```

Apply CP1 escalation for public repos. No auto-fix — manual review required.
