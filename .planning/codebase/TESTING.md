# Testing Patterns

**Analysis Date:** 2026-04-30

## Test Framework

**Runner:**
- No unit test framework (Jest, Vitest, Mocha, etc.) is present in the codebase
- Testing is done entirely through end-to-end CI smoke tests via GitHub Actions
- Config: `.github/workflows/compat-daily.yml`

**Assertion Style:**
- Bash `[[ condition ]] || { echo "::error::..."; exit 1; }` for CI assertions
- Log parsing to verify patch output: `grep -E 'Result: ...'`

**Run Commands:**
```bash
# Local simulation of the installer (end-to-end):
bash install.sh

# Installer with specific version:
bash install.sh --version 2.1.89

# Patcher in dry-run mode (no file writes):
node ~/.clawgod/patch.mjs --dry-run

# Patcher in verify mode (reports what would be applied):
node ~/.clawgod/patch.mjs --verify

# Patcher to revert to backup:
node ~/.clawgod/patch.mjs --revert

# Windows installer:
.\install.ps1
.\install.ps1 -DryRun
```

## Test File Organization

**Location:**
- No co-located unit test files (`*.test.*`, `*.spec.*`)
- All CI-level testing lives in `.github/workflows/compat-daily.yml`

**Structure:**
```
.github/
└── workflows/
    ├── compat-daily.yml    # End-to-end smoke test (daily + PR)
    ├── release.yml         # Release automation (tag-triggered)
    └── cache-cleanup-weekly.yml  # Cache maintenance
```

## CI Test Structure

**Daily Compatibility Smoke Test (`.github/workflows/compat-daily.yml`):**

The smoke job runs on `ubuntu-latest` only (macOS/Windows runners are 10x/2x more expensive for shared-tier). A Linux failure is treated as a cross-platform signal.

**Steps in order:**
1. Checkout repo
2. Set up Node.js 24
3. Set up Bun (canary channel — Anthropic ships with bleeding-edge Bun)
4. Cache `~/.npm` (restore from previous runs to speed up `npm pack`)
5. Install `ripgrep` (hard prerequisite for Claude Code's Grep tool)
6. Add `~/.local/bin` to `$GITHUB_PATH`
7. Print runtime versions
8. Run `install.sh` end-to-end, tee output to `/tmp/install.log`
9. Verify install artifacts exist in `~/.clawgod/`
10. Assert zero failed patches by parsing `install.log`
11. Smoke test `claude --version` — catches Bun CJS-wrapper panic
12. Publish verified Claude version to `badges` branch as JSON

**Triggers:**
- `schedule: '17 7 * * *'` — daily at 07:17 UTC
- `workflow_dispatch` — manual trigger
- `push` to `main` — on changes to `install.sh` or the workflow file
- `pull_request` — on changes to `install.sh` or the workflow file

## Assertions

**Artifact presence check:**
```bash
for f in cli.cjs cli.original.cjs patch.mjs extract-natives.mjs post-process.mjs .source-version; do
  test -e "$HOME/.clawgod/$f" || { echo "::error::missing ~/.clawgod/$f"; exit 1; }
done
```

**Patch success assertion (log parsing):**
```bash
line=$(grep -E 'Result: [0-9]+ applied, [0-9]+ skipped, [0-9]+ failed' /tmp/install.log | tail -1 || true)
[[ -n "$line" ]] || { echo "::error::patch.mjs result line missing from install.sh output"; exit 1; }
failed=$(echo "$line" | sed -E 's/.*skipped, ([0-9]+) failed.*/\1/')
[[ "$failed" -eq 0 ]] || { echo "::error::patch.mjs reported $failed failed patches"; exit 1; }
```

**Smoke test (Bun CJS panic detection):**
```bash
out=$(claude --version 2>&1)
rc=$?
if echo "$out" | grep -q "Expected CommonJS module to have a function wrapper"; then
  echo "::error::Bun CJS-wrapper panic — local Bun lags the embedded Bun"
  exit 1
fi
[[ $rc -eq 0 ]] || { echo "::error::claude --version exited $rc"; exit 1; }
```

## Patch Verification (patcher self-test)

The patcher (`patch.mjs`) has a built-in verification system separate from CI:

**Sentinel-based already-applied detection:**
- Each patch can declare a `sentinel` string — text that must NOT exist in a fully-patched file
- If pattern matches 0 times and sentinel is absent → patch already applied
- If pattern matches 0 times and sentinel still present → regex is stale (reports failure)

**Uniqueness enforcement:**
- `unique: true` patches fail if the pattern matches more than once (ambiguous — would corrupt unintended code)

**Conditional validation:**
- `validate: (match, code) => boolean` function narrows matches by context (e.g. checking nearby GrowthBook references)

**Index selection:**
- `selectIndex: 0` takes only the first match from multiple hits

**Patch modes:**
```bash
node patch.mjs             # Apply patches
node patch.mjs --dry-run   # Show what would change, write nothing
node patch.mjs --verify    # Report unpatched items, write nothing
node patch.mjs --revert    # Restore from .bak backup
```

## Failure Handling

**Scheduled run failure:**
- `compat-daily.yml` opens a GitHub issue labeled `compat-broken` on scheduled run failure
- Subsequent failures on an already-open issue add a comment (not a duplicate issue)
- Issue title format: `compat-daily: broke (claude <version>)`
- Uses `actions/github-script@v7` for issue management

**Badge publishing:**
- On success, the verified Claude version is written to `badges` branch as `claude-version.json`
- Badge is consumed by the README (shields.io endpoint) and the landing page (`web/src/main.ts`)

## Coverage

**Requirements:** No numeric coverage target — tests are integration-level only

**What is tested:**
- Full installer pipeline: download → extract → post-process → patch → launcher install
- All patches in `patches[]` array (zero-failed assertion)
- Bun compatibility with extracted `cli.original.cjs`
- `claude --version` happy path through the full wrapper chain

**What is NOT tested:**
- Individual parser functions in `extract-natives.mjs` (Mach-O, ELF, PE)
- Individual regex patterns in isolation
- Windows installer (`install.ps1`) — no Windows CI runners
- Web frontend (`web/src/main.ts`) — no frontend tests
- Config loading in `cli.cjs` wrapper

## Test Types

**Unit Tests:**
- Not present

**Integration Tests:**
- End-to-end installer test via `compat-daily.yml` on Linux only

**E2E Tests:**
- The smoke test (`claude --version`) is the closest equivalent — it exercises the full chain from launcher script through Bun runtime to `cli.original.cjs`

## Local Testing Patterns

**Installer dry-run equivalent:**
```bash
# The installer doesn't have a --dry-run flag, but patcher does:
bash install.sh 2>&1 | tee /tmp/install.log
node ~/.clawgod/patch.mjs --verify   # check patch state post-install
```

**Iterating on regex patches:**
```bash
# After editing patch.mjs in ~/.clawgod/:
node ~/.clawgod/patch.mjs --revert  # restore from .bak
node ~/.clawgod/patch.mjs           # re-apply updated patterns
claude --version                     # smoke test
```

**Re-patching after Claude Code upgrade:**
```bash
# This is the tested upgrade path:
claude update   # (patched to redirect to install.sh)
```

---

*Testing analysis: 2026-04-30*
