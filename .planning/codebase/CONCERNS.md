# Codebase Concerns

**Analysis Date:** 2026-04-30

## Tech Debt

**`--version` / `-Version` flags are parsed but silently ignored:**
- Issue: `install.sh` accepts `--version` and sets `VERSION="$2"`, and `install.ps1` accepts `-Version`. Neither variable is ever interpolated into the npm pack command — both scripts unconditionally fetch `@anthropic-ai/claude-code-<platform>@latest`. Users who pin a version (e.g. `install.sh --version 2.1.89`) get the latest anyway with no warning.
- Files: `install.sh` (lines 17–22, 183–185), `install.ps1` (lines 15–16, 162)
- Impact: Version pinning is a documented CLI feature (`.\install.ps1 -Version 2.1.89`) that does nothing. Debugging regressions by pinning to a known-good version is impossible without modifying the scripts manually.
- Fix approach: Pass `$VERSION` / `$Version` into the npm fetch call: `npm pack "$NPM_PKG@$VERSION"` (or validate that it equals `"latest"` and skip the parameter).

**`repatch.mjs` is installed but never invoked:**
- Issue: `install.sh` writes and makes executable `~/.clawgod/repatch.mjs` (a re-extract + re-patch helper), but the drift-detection block that called it was removed from `cli.cjs` because the `versions/` directory it relied on is no longer populated. The helper now exists at rest with no caller.
- Files: `install.sh` (lines 722–769, wrapper block at lines 782–795)
- Impact: Dead artifact shipped to every user's machine. Any future attempt to re-add drift detection must remember this file exists; mismatched states if `repatch.mjs` is updated without updating the detection logic in `cli.cjs`.
- Fix approach: Either re-wire the drift detection in `cli.cjs` to call `repatch.mjs`, or remove `repatch.mjs` from the installer and from the uninstall cleanup list.

**Stale error message references `$VERSIONS_DIR` after local detection was removed:**
- Issue: At `install.sh` line 210, the fallback error message reads `"Native Claude Code binary not found in $VERSIONS_DIR"`. But `VERSIONS_DIR` is only assigned later (line 1338) for a different purpose. At line 210, `$VERSIONS_DIR` expands to an empty string, producing a confusing error message.
- Files: `install.sh` (lines 209–214)
- Impact: If the npm fetch path fails entirely, the user sees `"Native Claude Code binary not found in "` (empty path) rather than a clear message.
- Fix approach: Replace the error with a static, accurate message like `"Failed to fetch native binary from npm registry."`.

**Inline heredoc extractor/patcher scripts duplicated across `install.sh` and `install.ps1`:**
- Issue: The full `extract-natives.mjs` (600+ lines), `post-process.mjs`, `patch.mjs`, `repatch.mjs`, and `cli.cjs` are embedded inline as heredocs in both `install.sh` and `install.ps1`. Any change to one requires manual sync in the other; there is no source-of-truth file for these scripts.
- Files: `install.sh` (lines 218–1230), `install.ps1` (lines 262–1180)
- Impact: High maintenance burden. A bug fix or feature added to the patcher in `install.sh` is silently absent from `install.ps1` until someone manually re-syncs.
- Fix approach: Maintain the scripts as separate committed files (e.g. `src/patch.mjs`, `src/extract-natives.mjs`), then build the installers with a script that inlines them at release time.

## Known Bugs

**Native module extraction failure is silently swallowed:**
- Symptoms: `extract-natives.mjs` for vendor `.node` modules is invoked with `|| true` — if it fails, the installer continues without any extracted native libraries. Features like Computer Use (which requires `computer-use-input.node`) silently break at runtime.
- Files: `install.sh` (line 666), `install.ps1` equivalent
- Trigger: Any platform where the embedded `.node` module heuristics fail (e.g. a new Bun build packs them differently).
- Workaround: Re-run the installer and observe individual extraction log lines; Computer Use errors appear only at feature invocation time.

**`patch.mjs` exit code not checked in `install.sh`:**
- Symptoms: If one or more non-optional patches fail (regex stale after an upstream change), `patch.mjs` prints a failure summary and exits non-zero, but `install.sh` line 1235 pipes the output through a formatter (`| while IFS= read...`) without capturing the exit status. `set -e` is not in effect for piped sub-commands under bash. The installer reports success and proceeds.
- Files: `install.sh` (line 1235)
- Trigger: Anthropic changes minified function signatures that break a regex in `patch.mjs`. CI catches this via `compat-daily.yml` assertion, but local installs proceed silently.
- Workaround: The CI `compat-daily.yml` asserts `failed == 0` and opens a GitHub issue automatically. End-user installs will leave features partially unpatched until the project ships a patch fix.

## Security Considerations

**No checksum or signature verification on downloaded npm tarball:**
- Risk: Both `install.sh` (via `npm pack`) and the inline `fetch.mjs` in `install.ps1` download the `@anthropic-ai/claude-code-<platform>` tarball from the npm registry over HTTPS but perform no integrity check (no `sha512` comparison against the registry metadata that is returned in the same response). A compromised npm package or a MITM with a valid certificate could deliver a malicious binary.
- Files: `install.sh` (lines 183–198), `install.ps1` `fetch.mjs` inline (lines 167–223)
- Current mitigation: HTTPS transport only; npm registry account controls.
- Recommendations: After fetching, compare the tarball SHA-512 against the `dist.integrity` field from the npm registry metadata JSON (which the Windows `fetch.mjs` already fetches). The metadata `dist.integrity` field is a SRI hash that can be verified with Node's built-in `crypto`.

**`provider.json` stores API key in plaintext:**
- Risk: `~/.clawgod/provider.json` contains `"apiKey": "<user_key>"` as plain text with default filesystem permissions (no explicit `chmod 600` applied by the installer).
- Files: `install.sh` (wrapper at lines 807–826), `~/.clawgod/provider.json` (runtime)
- Current mitigation: None beyond default user-owned home directory permissions.
- Recommendations: Set `chmod 600 ~/.clawgod/provider.json` after writing it, or document that users should do so.

**curl-pipe-bash / irm-iex install pattern:**
- Risk: The canonical install command is `curl -fsSL <url> | bash`. This pattern executes remotely fetched code without any verification. If the GitHub raw URL or release asset URL is hijacked, all installs are compromised. Same for `irm ... | iex` on Windows.
- Files: `README.md` (install instructions), `install.sh` (self-update redirect in `patch.mjs`, line 1003)
- Current mitigation: GitHub release asset integrity (GitHub controls the CDN), HTTPS pinned to `github.com`.
- Recommendations: Publish a `SHA256SUMS` file alongside release assets and document a verification step. Low priority for a developer tool but worth tracking.

## Performance Bottlenecks

**80 MB npm tarball downloaded on every `claude update`:**
- Problem: Because local detection is intentionally skipped (see INCIDENT_LOG 2026-04-29 comment), every upgrade path fetches the full `@anthropic-ai/claude-code-<platform>@latest` tarball (~60–90 MB compressed). `npm pack` populates `~/.npm` cache, but the cache key in `compat-daily.yml` is per-run-id so CI always re-downloads. For end users, the npm cache does help on repeated installs of the same version, but upgrades always re-download.
- Files: `install.sh` (lines 183–206), `.github/workflows/compat-daily.yml` (cache key at line ~72)
- Cause: Deliberate policy choice to avoid stale-source trap; the only optimization headroom is on the CI side.
- Improvement path: Use `restore-keys` more aggressively in CI, or switch the cache key to `npm-claude-${{ runner.os }}-<version>` so the same version is cached day-to-day and only re-downloaded on actual upstream bumps.

**`extract-natives.mjs` reads the entire Bun binary (~100+ MB) into memory:**
- Problem: `readFileSync(binaryPath)` at `extract-natives.mjs` line 577 loads the full standalone binary into a single `Buffer`. On a machine with < 512 MB free RAM this can cause OOM or significant swap pressure.
- Files: `install.sh` extractor heredoc (lines 218–647), equivalent in `install.ps1`
- Cause: Simpler implementation; the binary scanning uses `buf.indexOf` which requires the whole buffer.
- Improvement path: Stream-based scanning for the known fixed-offset anchors (CLI_PATH_MARKER, etc.) would reduce peak memory. Low priority unless reports emerge from low-RAM Linux hosts.

## Fragile Areas

**Regex patches in `patch.mjs` break whenever Anthropic changes minified identifiers:**
- Files: `install.sh` patcher heredoc (lines 867–1229), `install.ps1` equivalent
- Why fragile: Each patch encodes assumptions about minified variable name patterns (e.g. `function ([\w$]+)\(\)\{return"external"\}`) that are specific to Anthropic's current bundler output. Any change to minifier settings, function inlining, or code structure can silently break a patch if the sentinel string also changes.
- Safe modification: Add or update `sentinel` strings when changing a regex so "already applied" detection stays accurate. Run `node patch.mjs --verify` before and after.
- Test coverage: CI catches stale regexes via `compat-daily.yml` (asserts `failed == 0`), but only on Linux x64. macOS and Windows are not covered by the daily run.

**`CLI_PATH_MARKER` and `CLI_TAIL_MARKER` for `cli.js` extraction are hard-coded byte strings:**
- Files: `install.sh` extractor heredoc (lines 522–554), `install.ps1` equivalent
- Why fragile: The extraction relies on Bun embedding `file:///$bunfs/root/src/entrypoints/cli.js` as a marker string. If Anthropic changes the entry point path or Bun changes how it embeds the virtual FS path, primary anchor fails and the fallback anchor (`cli_after_main_complete`) must be relied on. If both fail, installation aborts with "Could not locate cli.js payload".
- Safe modification: Both anchors are currently independent. When updating, verify both independently with a known-good binary before shipping.
- Test coverage: Covered by `compat-daily.yml` smoke test (install + `claude --version`).

**Windows CI has zero automated coverage:**
- Files: `.github/workflows/compat-daily.yml`
- Why fragile: `install.ps1` is written, released, and documented but never executed in CI. Windows-specific code paths (the inline `fetch.mjs` Node script, `.cmd` launcher writing, `claude.exe` backup, PowerShell-specific proxy handling) can regress silently between releases.
- Safe modification: Any change to `install.ps1` requires manual Windows testing.
- Test coverage: None. `compat-daily.yml` uses `ubuntu-latest` only.

**macOS CI has zero automated coverage:**
- Files: `.github/workflows/compat-daily.yml`
- Why fragile: `compat-daily.yml` uses Linux only. macOS-specific Mach-O dylib extraction paths, `file` command usage, `stat -f%z` (BSD stat), and symlink backup logic (`ln -sf`) are untested in CI.
- Safe modification: Changes to dylib extraction or macOS launcher logic should be tested manually on both arm64 and x64 before merging.
- Test coverage: None in CI; covered only by maintainer manual testing.

**`post-process.mjs` overwrites `cli.original.js` in place (no atomic write):**
- Files: `install.sh` post-process heredoc (lines 677–710)
- Why fragile: The script reads `cli.original.js`, transforms it, writes `cli.original.cjs`, then `unlinkSync`s the source. If the process is interrupted (OOM kill, power loss) between write and unlink, both files may be in inconsistent states. Re-running the installer handles this because extraction re-creates `cli.original.js`.
- Safe modification: Low risk in practice; the installer is short-lived. No action required.

## Scaling Limits

**GitHub API rate limits affect web landing page stats:**
- Current capacity: 60 unauthenticated requests/hour per IP (GitHub REST API).
- Limit: A popular shared network (office, CI, university) hitting the page frequently will see `null` stats (stars, downloads show `—`).
- Scaling path: Add a GitHub token via the static site build process to raise the limit to 5000/hr, or cache the stats in the `badges` branch alongside the version JSON.

## Dependencies at Risk

**Dependency on Bun canary channel for runtime compatibility:**
- Risk: Anthropic builds `claude-code` with a canary Bun build (documented in CI comments). Users on stable Bun may encounter `"Expected CommonJS module to have a function wrapper"` panics. Canary is not available from most OS package managers (brew, apt, scoop). The project instructs users to run `bun upgrade --canary`, which is a non-standard upgrade path.
- Impact: Any Anthropic release that bumps to a new canary Bun breaks all users who cannot run `bun upgrade --canary` (e.g. managed systems, pinned environments).
- Migration plan: Bundle the specific canary Bun binary in the clawgod release, eliminating the user-side Bun version dependency. High cost (60 MB+ per platform per release) but removes the fragile external dependency.

**`@fontsource-variable/inter` and `@fontsource-variable/jetbrains-mono` in web build:**
- Risk: These packages are inlined at build time via `vite-plugin-singlefile` (fonts are base64 embedded into the HTML). If either package changes its woff2 encoding scheme or file layout, the build breaks silently or produces a broken font.
- Impact: Limited — fonts fall back to system fonts in browser; no functional breakage.
- Migration plan: Low priority. Pin to exact versions in `package.json` rather than `^5.2.8`.

## Missing Critical Features

**No rollback on partial install failure:**
- Problem: If extraction succeeds but patching fails (stale regex for a new Anthropic version), the installer has already overwritten the `claude` launcher binary. The user is left with a launcher that points to an unpatched `cli.original.cjs`. There is a `cli.original.cjs.bak` backup created by `patch.mjs` on first apply, and a `claude.orig` binary backup, but no automated rollback sequence.
- Blocks: Users cannot easily recover a working unpatched `claude` without manually running `install.sh --uninstall` and reinstalling the official binary.

**No version pinning or downgrade path:**
- Problem: The `--version` flag is documented and parsed but non-functional (see Tech Debt above). There is no supported way to install or revert to a specific Claude Code version once Anthropic removes old npm versions.

## Test Coverage Gaps

**No unit tests for `patch.mjs` regex patterns:**
- What's not tested: Individual regex patterns in `patch.mjs` are only validated against a live Anthropic binary in `compat-daily.yml`. There are no offline unit tests using fixture code snippets.
- Files: `install.sh` patcher heredoc (lines 882–1103)
- Risk: A regex change that breaks an unrelated patch in the same file will not be caught until the next daily run (or until a user reports it).
- Priority: High — these regexes are the core mechanism of the project.

**`install.ps1` has no automated test coverage:**
- What's not tested: The entire Windows installer path — binary fetch via inline `fetch.mjs`, `.node` extraction (PE parser), `patch.mjs` application, `.cmd` launcher creation, `claude.exe` backup.
- Files: `install.ps1`
- Risk: Any change to `install.ps1` can regress silently. Windows-specific bugs (PowerShell proxy, path separator, `.cmd` vs `.exe` precedence) are only caught by user reports.
- Priority: High — Windows is a supported platform with no safety net.

**No smoke test for `claude update` redirect:**
- What's not tested: The patched `claude update` action that redirects to `install.sh` is never executed in CI. The PowerShell base64-encoded redirect script embedded in `patch.mjs` is untested end-to-end.
- Files: `install.sh` patcher heredoc (lines 983–1008)
- Risk: A bug in the redirect (wrong base64 encoding, incorrect `spawnSync` call) leaves users unable to upgrade from within Claude Code.
- Priority: Medium.

---

*Concerns audit: 2026-04-30*
