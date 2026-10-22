# Summarize (`AGENTS.md`)

## Guardrails

- Multiple agents often work in this repository concurrently. If you see unfamiliar files or changes, do not touch them; list them at the end of your response.
- Never commit in `vendor/summarize` if present (treat it as read-only).
- Commits: use Conventional Commits (`committer "type: message" <files...>` when available).
- Before editing subsystem code, run `pnpm docs:list` and read any `docs/*.md` whose `read_when` hint matches your task.

## Workspace & Package Boundaries

pnpm workspace (`Node >=22`, `pnpm@10.25.0`):

- **Root (`@steipete/summarize`)**: CLI (`src/cli.ts`), local HTTP/SSE daemon (`src/daemon/` on `127.0.0.1:8787`), LLM execution (`src/llm/`), SQLite + media cache (`src/cache.ts`, `src/media-cache.ts`), and slide extraction (`src/slides/`). Re-exports `@steipete/summarize-core/content` and `@steipete/summarize-core/prompts` for backwards compatibility.
- **`packages/core` (`@steipete/summarize-core`)**: Programmatic library surface (`./content`, `./content/url`, `./prompts`, `./processes`, `./language`). Must stay free of CLI/TTY/daemon dependencies. Prefer importing from `@steipete/summarize-core` in apps.
- **`apps/chrome-extension` (`@steipete/summarize-chrome-extension`)**: WXT (Manifest V3) + Preact (`preact/compat` aliased as `react`) browser extension for Chrome Side Panel and Firefox Sidebar (`BROWSER=firefox`). Talks to the local daemon over `http://127.0.0.1:8787` with bearer token auth.

## Commands & Required Order

- **Dev CLI (no build needed)**: `pnpm summarize -- <args>` (or `pnpm s -- <args>`, runs `tsx src/cli.ts`).
- **Full build (order matters)**: `pnpm -s build`
  - Cleans `dist/` and `packages/core/dist/`, builds `packages/core` first, then `build:lib` (`tsc -p tsconfig.build.json`), then `build:cli` (`scripts/build-cli.mjs` ESM wrapper at `dist/cli.js`).
  - Note: `tsconfig.build.json` and `oxlint` require `packages/core/dist` types to exist unless resolved via Vitest aliases; build `@steipete/summarize-core` (`pnpm -C packages/core build`) before running standalone `pnpm typecheck` or `pnpm lint` on a clean checkout.
- **Gate (pre-PR / pre-release)**: `pnpm -s check`
  - Runs `pnpm format:check` (`oxfmt --check`) → `pnpm lint` (`oxlint --type-aware`) → `pnpm test:coverage` (Vitest with 75% v8 coverage threshold on lines/functions/branches/statements).
- **Format / lint fix**: `pnpm lint:fix` (runs `oxlint --fix` then `oxfmt --write`).
- **Bun standalone binary (macOS arm64)**: `pnpm build:bun` (or `pnpm build:bun:test` for binary + E2E smoke test in `dist-bun/`).

## Testing

- **All unit/integration tests**: `pnpm test` (`vitest run`)
- **Single test file**: `pnpm vitest run tests/<file>.test.ts` (all unit tests live in root `tests/**/*.test.ts`, including tests for `packages/core` and `apps/chrome-extension` helpers).
- **Vitest quirks**:
  - `vitest.config.ts` aliases `@steipete/summarize-core*` directly to `packages/core/src/*`, so unit tests do not require rebuilding `packages/core`.
  - `tests/setup.ts` sets `SUMMARIZE_DISABLE_LOCAL_WHISPER_CPP=1` globally.
  - Live tests (`tests/**/*.live.test.ts` and `tests/live/**`) are skipped by default; opt in with `SUMMARIZE_LIVE_TESTS=1` or `SUMMARIZE_LIVE_TEST=1` (plus `SUMMARIZE_LIVE_HEAVY_PODCASTS=1` where applicable) and required provider keys.
- **Extension E2E tests (Playwright)**:
  - Supported automated path: `pnpm -C apps/chrome-extension test:chrome` (builds `.output/chrome-mv3` and runs Chromium project).
  - First-time setup: `pnpm -C apps/chrome-extension exec playwright install chromium`.
  - Chromium E2E launches a persistent headful context minimized off-screen (`headless: false` for MV3 service worker support); set `SHOW_UI=1` to bring windows to front or `ALLOW_YOUTUBE_E2E=1` for live YouTube E2E tests.
  - Firefox Playwright extension tests are skipped by default due to `moz-extension://` navigation limitations; use `pnpm -C apps/chrome-extension test:firefox:force` (`ALLOW_FIREFOX_EXTENSION_TESTS=1`) only for explicit diagnostics.

## Extension + Daemon Workflow

- When rebuilding the extension + daemon, run **both** in order:
  1. `pnpm -C apps/chrome-extension build` (or `build:firefox` for `.output/firefox-mv3`)
  2. `pnpm -s summarize daemon restart` (verify with `pnpm -s summarize daemon status`)
- Dev daemon install (runs `src/cli.ts` via `tsx` without needing `dist/`): `pnpm summarize daemon install --token <TOKEN> --dev`.
- Daemon state/logs live in `~/.summarize/` (`config.json`, `daemon.json`, `cache.sqlite`, `cache/media/`, `logs/daemon.err.log`). Note: `launchd` does not inherit shell `PATH`; re-run `daemon install` after installing external media binaries (`ffmpeg`, `yt-dlp`, `tesseract`, `whisper-cli`) to refresh the daemon env snapshot.

## Versioning & Release

- **Lockstep versions**: Keep versions identical across all three places before releasing:
  1. `package.json` (`version`)
  2. `packages/core/package.json` (`version`)
  3. `src/version.ts` (`FALLBACK_VERSION` — required for the Bun-compiled binary)
  - Note: `apps/chrome-extension/wxt.config.ts` reads the extension manifest version from the root `package.json`, not `apps/chrome-extension/package.json`.
- **Publish order**: `@steipete/summarize-core` must be published to npm **before** `@steipete/summarize` (`scripts/release.sh` / `RELEASING.md`).

