# Codebase Structure

**Analysis Date:** 2026-04-30

## Directory Layout

```
clawgod/                          # Repo root — also GitHub Pages web root
├── install.sh                    # Primary installer (macOS / Linux)
├── install.ps1                   # Installer (Windows PowerShell)
├── index.html                    # Production landing page (built artifact, GitHub Pages)
├── bypass.png                    # OG image for landing page
├── CNAME                         # GitHub Pages custom domain (clawgod.0chen.cc)
├── README.md                     # English readme
├── README_JP.md                  # Japanese readme
├── README_ZH.md                  # Chinese readme
├── LICENSE                       # License
├── web/                          # Landing page source (Vite project)
│   ├── index.html                # Dev-time source HTML (never served directly)
│   ├── package.json              # web project manifest
│   ├── tsconfig.json             # TypeScript config for web
│   ├── vite.config.ts            # Vite build config (singlefile + deploy plugin)
│   ├── src/
│   │   ├── main.ts               # Progressive-enhancement JS entry point
│   │   └── styles/
│   │       ├── index.css         # CSS entrypoint (@import chain)
│   │       ├── tokens.css        # Design tokens + self-hosted fonts
│   │       ├── base.css          # CSS reset / base rules
│   │       ├── layout.css        # Page chrome, topbar, container
│   │       ├── hero.css          # Hero section styles
│   │       ├── sections.css      # Install widget, patches, how-it-works sections
│   │       └── responsive.css    # Media queries
│   └── public/                   # Static assets served as-is by Vite dev server
├── .github/
│   ├── workflows/
│   │   ├── compat-daily.yml      # Daily end-to-end install smoke test + badge publish
│   │   ├── release.yml           # Tag-triggered GitHub Release creation
│   │   └── cache-cleanup-weekly.yml  # Weekly npm cache eviction
│   └── ISSUE_TEMPLATE/
│       ├── bug_report.yml        # Bug report template
│       └── feature_request.yml   # Feature request template
└── .planning/
    └── codebase/                 # GSD codebase map documents
```

**Runtime artifacts (generated into `~/.clawgod/` at install time — not in repo):**
```
~/.clawgod/
├── extract-natives.mjs   # Binary extractor (Mach-O / ELF / PE parser)
├── post-process.mjs      # cli.js → cli.original.cjs converter
├── patch.mjs             # Regex patcher for cli.original.cjs
├── repatch.mjs           # Re-run full pipeline on binary upgrade
├── cli.cjs               # Runtime wrapper (sets env vars, require()s patched bundle)
├── cli.original.js       # Intermediate extracted JS (deleted after post-process)
├── cli.original.cjs      # Patched Claude Code bundle (CJS, runs under Bun)
├── cli.original.cjs.bak  # Pre-patch backup
├── .source-version       # Version string of the binary last patched
├── provider.json         # User config: API key, base URL, model, timeout
├── features.json         # GrowthBook feature flag overrides
└── vendor/               # Extracted .node NAPI modules (image-processor, etc.)
```

## Directory Purposes

**Repo root:**
- Purpose: Dual-role — installer distribution point and GitHub Pages web root.
- Contains: `install.sh`, `install.ps1` (released as GitHub Release assets), `index.html` (built artifact committed after `npm run build` in `web/`), static assets.
- Key files: `install.sh` (1,425 lines — entire install pipeline), `install.ps1` (1,336 lines — Windows equivalent), `index.html` (production single-file page).

**`web/`:**
- Purpose: Landing page source. Isolated Vite project; build output is committed back to repo root as `index.html`.
- Contains: TypeScript, CSS, Vite config; no backend or server-side code.
- Key files: `web/src/main.ts`, `web/src/styles/index.css`, `web/vite.config.ts`.

**`web/src/styles/`:**
- Purpose: CSS split by concern; imported in explicit order via `index.css`.
- Contains: Seven CSS files; `tokens.css` embeds self-hosted variable fonts (Inter, JetBrains Mono) as base64 data URIs via Vite's asset inlining.

**`web/public/`:**
- Purpose: Vite static pass-through directory (e.g., favicons, manifests).
- Generated: No. Committed: Yes.

**`.github/workflows/`:**
- Purpose: CI/CD — daily compat smoke test, release automation, cache hygiene.
- Key files: `compat-daily.yml` (most critical — catches upstream breakage; publishes `badges/claude-version.json`), `release.yml` (attaches `install.sh` + `install.ps1` to GitHub Releases on tag push).

**`.planning/codebase/`:**
- Purpose: GSD codebase map documents consumed by planning and execution commands.
- Generated: Yes (by `/gsd-map-codebase`). Committed: Yes.

## Key File Locations

**Entry Points:**
- `install.sh`: macOS/Linux install entrypoint (curl pipe target).
- `install.ps1`: Windows PowerShell install entrypoint.
- `web/src/main.ts`: Web landing page JS entry (loaded as `<script type="module">`).
- `web/vite.config.ts`: Vite project config and deploy plugin.

**Configuration:**
- `web/package.json`: Web project dependencies and scripts.
- `web/tsconfig.json`: TypeScript compiler options for `web/`.
- `.github/workflows/compat-daily.yml`: CI schedule, Bun canary setup, badge publish.
- `.github/workflows/release.yml`: Release asset upload on tag.

**Core Logic:**
- `install.sh` lines 218–648: Heredoc embedding `extract-natives.mjs` (binary extractor).
- `install.sh` lines 668–710: Heredoc embedding `post-process.mjs`.
- `install.sh` lines 771–861: Heredoc embedding `cli.cjs` (runtime wrapper).
- `install.sh` lines 865–1231: Heredoc embedding `patch.mjs` (patcher + all patch descriptors).
- `install.sh` lines 724–768: Heredoc embedding `repatch.mjs`.
- `install.sh` lines 1232–1236: Patch application step.
- `install.sh` lines 1237–1255: Default config creation (`features.json`, `provider.json`).
- `install.sh` lines 1291–1382: Launcher installation.

**Testing:**
- `.github/workflows/compat-daily.yml`: End-to-end integration test (install + `claude --version` smoke test + patch result assertion).
- No unit test framework; no test files in the repo source tree.

**Web styles:**
- `web/src/styles/tokens.css`: Design tokens (colors, type scale, spacing) + font embedding.
- `web/src/styles/index.css`: Import chain entrypoint (6 imports, order-sensitive).

## Naming Conventions

**Files:**
- Installer scripts: lowercase with extension: `install.sh`, `install.ps1`.
- Generated runtime scripts: `kebab-case.mjs` or `kebab-case.cjs`: `extract-natives.mjs`, `post-process.mjs`, `patch.mjs`, `repatch.mjs`, `cli.cjs`.
- Config files: `camelCase.json`: `provider.json`, `features.json`.
- CSS files: `kebab-case.css`: `tokens.css`, `hero.css`, `sections.css`, `responsive.css`.
- Web source: `main.ts`, `index.css` (conventional Vite entry names).
- Workflow files: `kebab-case.yml` with suffix describing trigger: `compat-daily.yml`, `release.yml`, `cache-cleanup-weekly.yml`.

**Directories:**
- Source: `web/`, `web/src/`, `web/src/styles/` — lowercase.
- Meta: `.github/`, `.planning/` — dot-prefix for tooling directories.

**CSS classes:**
- BEM-influenced kebab-case: `.install-tab`, `.install-panel`, `.copy-btn`, `.hero-meta`, `.topbar-inner`, `.stat-num`.
- State modifiers as flat classes: `.active`, `.scrolled`, `.copied`, `.failed`, `.loading`, `.idle`.

**Patch descriptors:**
- `name` field: human-readable, title-case with feature description: `'USER_TYPE → ant'`, `'Agent Teams always enabled'`, `'Voice Mode enable (bypass GrowthBook kill)'`.

## Where to Add New Code

**New patch (feature unlock):**
- Add a new entry to the `patches` array inside the `patch.mjs` heredoc in `install.sh`, starting at line ~882.
- Follow the patch descriptor schema: `{ name, pattern, replacer, sentinel?, unique?, optional? }`.
- Include a `sentinel` whenever possible so the idempotency check can distinguish "already applied" from "regex stale".
- Test by running `install.sh` on a fresh environment and confirming the CI `Assert all patches applied` step reports `0 failed`.

**New environment variable injection:**
- Add to the `cli.cjs` heredoc in `install.sh` (lines 771–861), inside the block that sets `process.env.*` before `require('./cli.original.cjs')`.

**New feature flag default:**
- Add the key/value to the `features.json` heredoc written at `install.sh` lines 1240–1254.

**New web landing page section:**
- Add HTML to `web/index.html` (dev source).
- Add corresponding CSS to the appropriate file in `web/src/styles/` (likely `sections.css` for content sections, `hero.css` for above-the-fold).
- Run `cd web && npm run build` to rebuild `index.html` at the repo root.

**New CI workflow:**
- Add a `.yml` file to `.github/workflows/`. Follow the naming pattern `<purpose>-<trigger>.yml`.
- Include `FORCE_JAVASCRIPT_ACTIONS_TO_NODE24: "true"` as a top-level `env` (project policy).

**Utilities / shared shell helpers:**
- Shell helpers (coloured output, etc.) live inline in `install.sh` (lines 28–38). Add new helpers in the same block.

## Special Directories

**`web/public/`:**
- Purpose: Vite static asset pass-through.
- Generated: No.
- Committed: Yes (if populated).

**`~/.clawgod/vendor/`** (runtime, not in repo):
- Purpose: Extracted platform-native NAPI `.node` modules.
- Generated: Yes (at install time by `extract-natives.mjs`).
- Committed: No.

**`.planning/`:**
- Purpose: GSD planning documents.
- Generated: Yes (by GSD commands).
- Committed: Yes.

---

*Structure analysis: 2026-04-30*
