# Architecture

**Analysis Date:** 2026-04-30

## Pattern Overview

**Overall:** Shell-script installer + embedded multi-stage pipeline, with a static landing page as a parallel concern.

**Key Characteristics:**
- The entire patcher, extractor, post-processor, wrapper, and launcher are all generated as inline heredoc scripts within `install.sh` and written to `~/.clawgod/` at install time — there is no separate source tree for these runtime artifacts.
- Patches are regex-based and version-agnostic: each patch carries a `pattern`, `replacer`, and optional `sentinel` for idempotency detection rather than relying on line-number or string-literal matches.
- The runtime entrypoint is `~/.clawgod/cli.cjs` (a CJS wrapper), which configures environment variables from `provider.json` / `features.json` and then `require()`s `cli.original.cjs` (the extracted and patched Claude Code source).
- The web landing page (`web/`) is an entirely separate concern that gets built into a single self-contained `index.html` and copied to the repo root for GitHub Pages.

## Layers

**Installer (`install.sh` / `install.ps1`):**
- Purpose: Orchestrator for the entire install pipeline — fetches source binary, writes all runtime scripts, applies patches, installs launcher.
- Location: `install.sh` (macOS/Linux), `install.ps1` (Windows PowerShell)
- Contains: Prerequisite checks, npm tarball fetcher, heredoc-embedded sub-scripts, launcher writer, PATH check.
- Depends on: Node.js ≥18, Bun (canary), ripgrep, npm
- Used by: End users via `curl … | bash` or PowerShell `irm … | iex`

**Extractor (`extract-natives.mjs` — generated at install time):**
- Purpose: Parse the Bun standalone binary (Mach-O / ELF / PE), extract `cli.js` text and embedded `.node` NAPI modules.
- Location: Written by `install.sh` to `~/.clawgod/extract-natives.mjs`
- Contains: Binary format parsers (Mach-O, ELF, PE), `cli.js` text extractor (two-anchor strategy: bunfs path marker + fallback `cli_after_main_complete` tail scan).
- Depends on: Node.js `fs` built-ins only; no npm packages.
- Used by: `install.sh` directly after writing, and by `repatch.mjs` on re-patch.

**Post-processor (`post-process.mjs` — generated at install time):**
- Purpose: Rewrite extracted `cli.original.js` for Bun CJS runtime compatibility.
- Location: Written by `install.sh` to `~/.clawgod/post-process.mjs`
- Contains: Rewrites `/$bunfs/root/X.node` paths to point at extracted vendor modules; rewrites build-time `/home/runner/.../*.ts` asset URLs to `__filename`; wraps IIFE with CJS invocation; saves as `cli.original.cjs`.
- Depends on: Node.js `fs` built-ins; reads `~/.clawgod/cli.original.js`, writes `~/.clawgod/cli.original.cjs`.
- Used by: `install.sh` pipeline, `repatch.mjs`.

**Patcher (`patch.mjs` — generated at install time):**
- Purpose: Apply regex-based feature unlocks and restriction removals to `cli.original.cjs`.
- Location: Written by `install.sh` to `~/.clawgod/patch.mjs`
- Contains: Ordered `patches` array (each with `name`, `pattern`, `replacer`, optional `sentinel`/`unique`/`selectIndex`/`validate`/`optional` fields); patch application loop with idempotency detection via sentinel strings.
- Depends on: Node.js `fs` built-ins; reads and overwrites `~/.clawgod/cli.original.cjs`.
- Used by: `install.sh` after post-process, `repatch.mjs`.

**Re-patch helper (`repatch.mjs` — generated at install time):**
- Purpose: Re-run the full extract → post-process → patch pipeline against a freshly downloaded binary.
- Location: Written by `install.sh` to `~/.clawgod/repatch.mjs`
- Contains: Orchestrates `extract-natives.mjs` (twice: `--cli-js` and native modules), `post-process.mjs`, `patch.mjs` via `spawnSync`.
- Used by: Invoked manually or by `claude update` redirect.

**Wrapper (`cli.cjs` — generated at install time):**
- Purpose: Runtime entry point. Reads `provider.json` and `features.json`, sets environment variables, performs one-time `.claude.json` migration, then `require()`s `cli.original.cjs`.
- Location: Written by `install.sh` to `~/.clawgod/cli.cjs`; executed by Bun.
- Key env vars set: `ANTHROPIC_API_KEY`, `ANTHROPIC_BASE_URL`, `ANTHROPIC_MODEL`, `ANTHROPIC_SMALL_FAST_MODEL`, `API_TIMEOUT_MS`, `CLAUDE_CODE_DISABLE_NONESSENTIAL_TRAFFIC`, `DISABLE_INSTALLATION_CHECKS`, `USE_BUILTIN_RIPGREP`, `CLAUDE_INTERNAL_FC_OVERRIDES`.
- Used by: The `claude` / `clawgod` shell launchers via `bun ~/.clawgod/cli.cjs "$@"`.

**Launcher (shell script — written to `$PATH` at install time):**
- Purpose: Thin shell shim that invokes `bun ~/.clawgod/cli.cjs "$@"`.
- Location: Written to `$(dirname $(which claude))/claude`, `~/.local/bin/claude`, and `~/.local/bin/clawgod`.
- Depends on: Bun binary path baked at install time; falls back to `command -v bun`.

**Web landing page (`web/`):**
- Purpose: Static marketing/download site served via GitHub Pages.
- Location: `web/src/main.ts` (progressive-enhancement JS), `web/src/styles/` (CSS), `web/index.html` (dev source), root `index.html` (production build output).
- Build: Vite + `vite-plugin-singlefile` inlines all CSS, JS, and fonts into a single `index.html`. A custom `deploy-to-repo-root` Vite plugin copies `web/dist/index.html` → `index.html` after each build.
- Hydration: Fetches `badges/claude-version.json` from the `badges` branch for the verified-version pill; fetches GitHub API for star and download counts.

## Data Flow

**Install pipeline:**

1. User runs `curl … | bash` or downloads `install.sh` and runs it directly.
2. `install.sh` checks prerequisites (Node ≥18, Bun canary, ripgrep).
3. `install.sh` runs `npm pack @anthropic-ai/claude-code-<platform>@latest` into a tmpdir and extracts the native Bun standalone binary.
4. `install.sh` writes `extract-natives.mjs`, `post-process.mjs`, `repatch.mjs`, `patch.mjs`, `cli.cjs` to `~/.clawgod/` via heredocs.
5. `install.sh` invokes `node extract-natives.mjs <binary> ~/.clawgod --cli-js` → writes `~/.clawgod/cli.original.js`.
6. `install.sh` invokes `node extract-natives.mjs <binary> ~/.clawgod/vendor` → writes extracted `.node` NAPI modules.
7. `install.sh` invokes `node post-process.mjs` → rewrites `cli.original.js` and saves as `cli.original.cjs`.
8. `install.sh` invokes `node patch.mjs` → applies regex patches in-place on `cli.original.cjs`.
9. `install.sh` creates default `provider.json` and `features.json` in `~/.clawgod/` if not present.
10. `install.sh` sanity-checks: `bun ~/.clawgod/cli.cjs --version` must not panic.
11. `install.sh` writes the `claude` / `clawgod` shell launchers to `$PATH`.

**Runtime (user runs `claude`):**

1. Shell launcher executes `bun ~/.clawgod/cli.cjs "$@"`.
2. `cli.cjs` migrates any stray `.claude.json` from `~/.clawgod/` to `~/`.
3. `cli.cjs` reads `~/.clawgod/provider.json`; sets `ANTHROPIC_*` and `API_TIMEOUT_MS` env vars.
4. `cli.cjs` reads `~/.clawgod/features.json`; sets `CLAUDE_INTERNAL_FC_OVERRIDES` env var.
5. `cli.cjs` calls `require('./cli.original.cjs')` — the patched Claude Code bundle runs under Bun.

**Update flow:**

- `claude update` is patched (via `patch.mjs`) to redirect to `install.sh` rather than Anthropic's own update mechanism, ensuring the re-patch pipeline always runs against the freshest npm release.

**CI badge flow:**

- `.github/workflows/compat-daily.yml` runs `install.sh` end-to-end on `ubuntu-latest` daily.
- On success, it force-pushes `claude-version.json` to the `badges` branch.
- The landing page's `main.ts` fetches this JSON to display the verified Claude version pill.

## Key Abstractions

**Patch descriptor object:**
- Purpose: Declarative description of a single code transformation.
- Pattern:
  ```javascript
  {
    name: 'Human-readable label',
    pattern: /regex/g,
    replacer: (match, ...groups) => replacement,
    sentinel: 'string that should be absent after patch',  // optional
    unique: true,      // fail if >1 match
    selectIndex: 0,    // apply only to Nth match
    validate: (match, code) => boolean,  // contextual guard
    optional: true,    // skip silently if 0 matches
  }
  ```
- Location: `patch.mjs` (inside `install.sh` heredoc, `~/.clawgod/patch.mjs` at runtime), `install.sh` lines 882–1100+.

**Binary extractor (format-agnostic):**
- Purpose: Scan Bun standalone executables for embedded `.node` dylibs and the `cli.js` text payload using format-specific magic bytes.
- Examples: `parseMachODylib()`, `parseELFSharedObject()`, `parsePEDll()`, `extractCliJs()` in `extract-natives.mjs`.
- Pattern: Magic-byte scan (`buf.indexOf(magicBytes, off)`) → structural parse → offset + size → `buf.slice(offset, offset + size)`.

**Provider config (`provider.json`):**
- Purpose: User-level override for API key, base URL, model selection, and timeout.
- Location: `~/.clawgod/provider.json` (created with defaults on first install).
- Consumed by: `cli.cjs` at startup.

**Feature flags (`features.json`):**
- Purpose: GrowthBook feature flag overrides injected via `CLAUDE_INTERNAL_FC_OVERRIDES` env var.
- Location: `~/.clawgod/features.json` (created with opinionated defaults on first install).
- Consumed by: `cli.cjs` at startup; read by the patched GrowthBook function in `cli.original.cjs`.

## Entry Points

**User install (macOS/Linux):**
- Location: `install.sh`
- Triggers: `curl … | bash` or direct `bash install.sh`
- Responsibilities: Full install pipeline; uninstall with `--install` arg; version pinning with `--version X`.

**User install (Windows):**
- Location: `install.ps1`
- Triggers: `irm … | iex` in PowerShell
- Responsibilities: Equivalent install pipeline for Windows/PowerShell.

**Runtime entry (user shell):**
- Location: `~/.local/bin/claude` or `~/.local/bin/clawgod` (generated launchers)
- Triggers: User types `claude` or `clawgod`
- Responsibilities: Exec `bun ~/.clawgod/cli.cjs "$@"`.

**Web build:**
- Location: `web/vite.config.ts`
- Triggers: `npm run build` inside `web/`
- Responsibilities: Bundle `web/src/main.ts` + CSS into single-file `web/dist/index.html`; copy to repo root `index.html`.

**CI pipeline:**
- Location: `.github/workflows/compat-daily.yml`, `.github/workflows/release.yml`, `.github/workflows/cache-cleanup-weekly.yml`
- Triggers: Daily cron, tag push, PR/push to `install.sh`.

## Error Handling

**Strategy:** Fail-fast with `set -e` in `install.sh`; explicit `exit 1` with coloured `warn` messages for every detectable failure condition. Patcher uses a structured result counter (applied / skipped / failed) and the CI workflow asserts `failed == 0`.

**Patterns:**
- Prerequisite failures: Print human-readable install instructions and `exit 1` immediately.
- Patch failures: Increment `failed` counter; log sentinel-based diagnosis; CI asserts the count.
- Idempotent sentinel check: If patch `pattern` matches 0 results, check whether the sentinel string is absent (already applied) or still present (regex is stale → fail).
- Bun CJS panic detection: `install.sh` sanity-checks `bun cli.cjs --version` output for the known panic string before installing the launcher.
- Binary format unknown: `extract-natives.mjs` exits with code 1 and a clear message.

## Cross-Cutting Concerns

**Logging:** `install.sh` uses coloured `info()` / `warn()` / `dim()` bash functions; generated Node scripts use `console.log` with emoji status prefixes (`✅`, `❌`, `⚠️`, `⏭`).

**Validation:** Patch descriptor `validate` callbacks provide context-aware guards; `unique` and `selectIndex` fields guard against over-matching.

**Authentication:** No auth in the tool itself. API key optionally stored in `~/.clawgod/provider.json` and injected as `ANTHROPIC_API_KEY` into the Claude Code process environment.

---

*Architecture analysis: 2026-04-30*
