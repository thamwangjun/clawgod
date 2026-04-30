# Coding Conventions

**Analysis Date:** 2026-04-30

## Language Mix

This codebase uses three languages with distinct conventions per layer:

- **Bash** (`install.sh`) — 1,425 lines, primary installer and patcher host
- **PowerShell** (`install.ps1`) — 1,336 lines, Windows installer parity
- **TypeScript** (`web/src/main.ts`, `web/vite.config.ts`) — landing page frontend
- **Node.js ESM** (`.mjs` scripts generated at install time into `~/.clawgod/`) — patch/extract tooling

## Naming Patterns

**Shell Variables (Bash):**
- SCREAMING_SNAKE_CASE for script-level constants: `CLAWGOD_DIR`, `BIN_DIR`, `VERSION`, `NATIVE_BIN`, `NATIVE_BIN_LABEL`
- lowercase for local loop variables: `off`, `cand`, `sz`, `dir`, `target`
- Env vars passed to child processes use ANTHROPIC prefix: `ANTHROPIC_API_KEY`, `ANTHROPIC_BASE_URL`, `ANTHROPIC_MODEL`

**PowerShell Variables:**
- PascalCase for script-level variables: `$ClawDir`, `$BinDir`, `$NativeBin`, `$NativeBinLabel`, `$BunBin`
- PascalCase for function names: `Write-OK`, `Write-Err`, `Write-Warn`, `Write-Dim`

**TypeScript/JavaScript Functions:**
- camelCase for functions: `copyText`, `formatCount`, `setStat`, `archName`, `platformSuffix`
- camelCase for variables: `topbar`, `meta`, `dot`, `text`
- UPPER_SNAKE_CASE for module-level constants: `REPO`, `BADGES_JSON`, `KNOWN_MODULES`, `CLI_PATH_MARKER`, `MH_MAGIC_64`
- camelCase for object properties: `schemaVersion`, `stargazers_count`, `download_count`

**Patch descriptors (in `patch.mjs`):**
- `name`: human-readable string in imperative form: `'USER_TYPE → ant'`, `'Agent Teams always enabled'`
- `pattern`: RegExp with `g` flag, capturing groups with `[\w$]+` for minified identifiers
- `replacer`: arrow function receiving full match + capture groups
- Optional fields: `sentinel`, `optional`, `unique`, `validate`, `selectIndex`

## Code Style

**Formatting (Bash):**
- 2-space indentation throughout
- Section dividers use box-drawing dashes: `# ─── Section name ────`
- Helper functions `info()`, `warn()`, `dim()` for all user-facing output — never raw `echo`
- Heredoc pattern for embedding Node.js scripts: `cat > "$FILE" << 'EOF'`
- `set -e` at top of `install.sh` for fail-fast behavior

**Formatting (PowerShell):**
- 4-space indentation
- Section dividers match Bash style: `# ─── Section name ────`
- Analogous helper functions: `Write-OK`, `Write-Err`, `Write-Warn`, `Write-Dim`
- `$ErrorActionPreference = "Stop"` at top for fail-fast behavior

**Formatting (TypeScript):**
- 2-space indentation
- Single quotes for strings
- Semicolons present
- `strict: true`, `noUnusedLocals: true`, `noUnusedParameters: true` in `web/tsconfig.json`
- Section dividers in comments: `/* ─── Section name ───── */`

**Formatting (Node.js `.mjs`):**
- 2-space indentation
- Single quotes for strings
- Template literals for multiline output
- Section dividers match other files: `// ─── Section name ─────`

## Import Organization

**TypeScript (`web/src/main.ts`):**
- No imports — the file is a pure DOM manipulation script; CSS is loaded via `<link>` in HTML, not imported through JS (intentional, documented in the file header)

**Node.js (generated `.mjs` files):**
- Named imports from Node built-ins only: `import { readFileSync, writeFileSync, ... } from 'fs'`
- No external dependencies; tooling is intentionally self-contained

**Vite config (`web/vite.config.ts`):**
```typescript
import { defineConfig } from 'vite';
import { viteSingleFile } from 'vite-plugin-singlefile';
import { copyFileSync } from 'node:fs';
import { resolve } from 'node:path';
```
- Third-party first, then Node built-ins prefixed with `node:`

## Error Handling

**Bash:**
- `set -e` ensures any unhandled non-zero exit aborts the script
- Errors printed via `warn()` helper with red `✗` prefix before `exit 1`
- Graceful degradation with `|| true` where failures are expected (e.g. `hash -r 2>/dev/null`)
- Pre-flight checks (node version, bun presence, rg presence) fail early with actionable messages
- Pattern: check condition → print descriptive error → `exit 1`

```bash
if ! command -v node &>/dev/null; then
  warn "Node.js is required (>= 18) for the patcher. Install from https://nodejs.org"
  exit 1
fi
```

**PowerShell:**
- `$ErrorActionPreference = "Stop"` at top — terminates on any cmdlet error
- Mirrors Bash pattern: check → `Write-Err` → `exit 1`
- Exit codes propagated via `$LASTEXITCODE` check after `& node ...` calls

**Node.js (`.mjs` scripts):**
- Guard clauses at top of `main()` with `process.exit(1)` on invalid args
- Binary format failures return `null` from parsers (never throw)
- File I/O errors allowed to throw naturally (uncaught = process exit with stack trace)
- `try { } catch {}` used in wrapper (`cli.cjs`) for non-critical operations like config parse and file migration

```javascript
try {
  const raw = JSON.parse(readFileSync(configFile, 'utf8'));
  config = { ...defaultConfig, ...raw };
} catch {}
```

**TypeScript (web):**
- `async/await` with `try { } catch { /* fall through */ }` for network requests
- Graceful degradation: failed fetches leave UI in static fallback state — no visible errors to user

```typescript
try {
  const res = await fetch(BADGES_JSON, { cache: 'no-store' });
  if (!res.ok) throw new Error(`HTTP ${res.status}`);
  // ... update UI
} catch {
  /* keep static fallback */
}
```

## Logging

**Bash installer output:**
- `info()` → green `✓` prefix — success/confirmation
- `warn()` → red `✗` prefix — errors and final notices requiring action
- `dim()` → dim text — progress steps, secondary info

**Node.js patcher output:**
- `✅` for applied patches
- `⏭` for skipped (not present / no-op)
- `❌` for failed (regex stale)
- `⚠️` for ambiguous (multiple matches when uniqueness required)
- Summary line: `Result: N applied, N skipped, N failed`

**TypeScript (web):**
- No runtime logging; `console.log` used only in `vite.config.ts` with `// eslint-disable-next-line no-console` comment, acknowledging it's a build-time side effect

## Comments

**Documentation Style:**
- Block comments at top of files describe purpose and usage
- Section headers use consistent visual dividers: `# ─── Name ─────────────`
- Inline comments explain non-obvious decisions: binary format constants, regex escape choices, platform quirks
- Long explanatory notes for policy decisions (especially around version detection) are placed near the relevant code rather than in separate docs

**When to Comment:**
- All regex patterns in `patches` array have companion comments explaining version-specific behavior and structural changes between Claude Code releases
- Platform-specific quirks documented inline (Bun canary vs stable, PS 5.1 vs 7, musl detection)
- Removed/refactored logic gets a dated `NOTE:` comment explaining why, e.g. `// Note: drift detection removed — see install.sh wrapper for full notes.`

```bash
# Detection policy: ALWAYS pull from the npm registry @latest.
#
# Earlier versions of this script also probed local `node_modules` roots
# ... (full rationale inline)
# See INCIDENT_LOG 2026-04-29 entry.
```

## Function Design

**Bash:**
- Short single-purpose functions for output: `info()`, `warn()`, `dim()`
- One `write_launcher()` function for write-then-chmod pattern
- Everything else is sequential top-to-bottom script flow

**PowerShell:**
- Output helpers as simple one-liners: `function Write-OK($msg) { Write-Host "  ✓ $msg" -ForegroundColor Green }`
- No complex PowerShell functions; logic is inline sequential

**Node.js (`.mjs`):**
- Pure parser functions return `null` on failure, data object on success
- Scan functions (`extractMachODylibs`, `extractELFSharedObjects`, `extractPEDlls`) return arrays
- Single `main()` function at bottom is the entry point, called immediately
- `identifyDylib()` uses two-stage fallback: install name first, body scan second

**TypeScript (web):**
- Small focused functions: `copyText(text)` → `Promise<boolean>`, `formatCount(n)` → `string`, `setStat(id, n)` → `void`
- IIFEs for async blocks that run at module load: `(async () => { ... })()`

## Module Design

**Exports:**
- No module exports anywhere — all scripts are top-level executables or self-contained IIFEs
- `web/src/main.ts` compiled by Vite with `inlineDynamicImports: true`; output is a single inlined script tag

**Generated Scripts:**
- `extract-natives.mjs`, `post-process.mjs`, `repatch.mjs`, `patch.mjs` are written by the installer into `~/.clawgod/` at install time — they are embedded as heredocs in `install.sh` and `install.ps1`
- `cli.cjs` (the runtime wrapper) is also generated — runs under Bun, not Node

**No Barrel Files:**
- Not applicable; project is not a library

---

*Convention analysis: 2026-04-30*
