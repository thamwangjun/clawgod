# Technology Stack

**Analysis Date:** 2026-04-30

## Languages

**Primary:**
- Bash (POSIX + bash 5+) - macOS/Linux installer (`install.sh`)
- PowerShell 5.1+ - Windows installer (`install.ps1`)
- JavaScript (ES2022 / ESM) - All runtime scripts: patcher, extractor, wrapper (`patch.mjs`, `extract-natives.mjs`, `post-process.mjs`, `repatch.mjs`)
- TypeScript 5.6+ - Landing page frontend (`web/src/main.ts`)
- CSS - Landing page styles (`web/src/styles/`)

**Secondary:**
- CommonJS (CJS) - Runtime wrapper that runs under Bun (`~/.clawgod/cli.cjs` — generated at install time)

## Runtime

**Environment:**
- Node.js >= 18 (hard requirement; used for patcher scripts `patch.mjs`, `extract-natives.mjs`, `post-process.mjs`)
- Bun (canary channel preferred; used as the Claude Code runtime) — installed automatically if absent
- ripgrep (`rg`) — hard runtime prerequisite for Claude Code's Grep tool

**Execution model:**
- Installer scripts write all tooling files to `~/.clawgod/` at install time
- Bun executes `~/.clawgod/cli.cjs` (the wrapper), which then `require()`s `cli.original.cjs` (patched)
- Shell launcher at `~/.local/bin/claude` (bash) or `~/.local/bin/claude.cmd` (Windows) invokes `bun cli.cjs`

**Package Manager:**
- npm — used only during install to `npm pack` the Anthropic platform tarball from the registry
- Lockfile: `web/package-lock.json` (present for the web project)
- No package manager for the installer itself; all deps are shell-resolved at runtime

## Frameworks

**Core:**
- None (installer is pure shell + Node.js scripts)

**Frontend (web landing page):**
- Vite ^7.0.0 — build tool (`web/vite.config.ts`)
- `vite-plugin-singlefile` ^2.0.3 — inlines all CSS/JS into a single `index.html` for GitHub Pages

**Testing:**
- Not detected (no test framework configured)

**Build/Dev:**
- Vite dev server on port 5173 (`web/vite.config.ts`)
- TypeScript compiler via Vite (not standalone tsc)

## Key Dependencies

**Web project (`web/package.json`):**
- `@fontsource-variable/inter` ^5.2.8 — Self-hosted Inter variable font, base64-inlined into CSS
- `@fontsource-variable/jetbrains-mono` ^5.2.8 — Self-hosted JetBrains Mono, base64-inlined into CSS
- `typescript` ^5.6.3 — TypeScript compiler (devDependency)
- `vite` ^7.0.0 — Build tool (devDependency)
- `vite-plugin-singlefile` ^2.0.3 — Single-file output plugin (devDependency)

**Runtime dependencies (resolved at install time, not declared in package.json):**
- `@anthropic-ai/claude-code-<platform>` — Anthropic's native Claude Code binary fetched from npm registry (platform: `darwin-arm64`, `darwin-x64`, `linux-arm64`, `linux-x64`, `linux-arm64-musl`, `win32-x64`, `win32-arm64`)
- Bun runtime (canary) — fetched from `bun.sh/install` if not present
- ripgrep — must be installed via system package manager

## Configuration

**Environment:**
- No `.env` files used; all runtime config lives in `~/.clawgod/provider.json` (user-managed)
- `~/.clawgod/provider.json` — API key, base URL, model, smallModel, timeoutMs
- `~/.clawgod/features.json` — GrowthBook feature flag overrides (JSON object)
- `~/.clawgod/.source-version` — Tracks installed Claude Code version for upgrade detection

**Key env vars injected by wrapper (`~/.clawgod/cli.cjs`):**
- `ANTHROPIC_API_KEY` — from `provider.json.apiKey`
- `ANTHROPIC_BASE_URL` — from `provider.json.baseURL` (default: `https://api.anthropic.com`)
- `ANTHROPIC_MODEL` — from `provider.json.model`
- `ANTHROPIC_SMALL_FAST_MODEL` — from `provider.json.smallModel`
- `ANTHROPIC_AUTH_TOKEN` — set for non-Anthropic base URLs
- `API_TIMEOUT_MS` — from `provider.json.timeoutMs` (default: 3,000,000 ms)
- `CLAUDE_CODE_DISABLE_NONESSENTIAL_TRAFFIC=1` — always set
- `DISABLE_INSTALLATION_CHECKS=1` — always set
- `USE_BUILTIN_RIPGREP=1` — always set (use system rg, not bundled)
- `CLAUDE_INTERNAL_FC_OVERRIDES` — set from `features.json` if present

**Build:**
- Web build: `web/tsconfig.json` — target ES2022, strict mode, isolatedModules
- Vite config: `web/vite.config.ts` — single-file output, custom post-build copy to repo root

## Platform Requirements

**Development:**
- Node.js >= 18
- Bun (canary recommended)
- ripgrep
- npm (for fetching Anthropic platform packages)

**Production (installed user system):**
- macOS (arm64, x64) or Linux (arm64, x64, musl variants) or Windows (x64, arm64)
- Installed to: `~/.clawgod/` (all tooling), `~/.local/bin/` (launchers)
- Deployed web landing page: GitHub Pages at `clawgod.0chen.cc` (single `index.html` in repo root)

---

*Stack analysis: 2026-04-30*
