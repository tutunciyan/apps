# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

A static, no-build, client-side-only web app: an Azure DevOps Pull Requests report/dashboard. There is no server, no package manager, and no test suite — it's plain HTML/CSS/JS served directly from disk or any static file host.

- `pull-requests/index.html` — page shell (PAT gate, loading pane, filter bar, report container).
- `pull-requests/main.js` — all logic: Azure DevOps REST calls, data shaping, rendering, filtering/sorting, URL state sync. Single IIFE, no modules/bundler/framework.
- `pull-requests/main.css` — all styling.
- `PullRequests.html` — legacy path kept for old bookmarks/links; it's a pure JS redirect (`window.location.replace`) into `pull-requests/index.html`, forwarding the query string and hash. Keep this redirect working when renaming/moving the app further.

## Running / testing

No install or build step. Open `pull-requests/index.html` directly in a browser, or serve the repo root with any static file server (needed if testing `?pat=` URL params or PAT persistence, since `file://` origins can behave inconsistently with `sessionStorage`/fetch). There are no automated tests or linters configured — verify changes manually in a browser.

## Architecture

Everything lives in `pull-requests/main.js`, organized top-to-bottom into the sections below (search for the `==== ... ====` comment banners):

1. **Configuration** — reads `organization`, `project`, `status`, `groupBy`, `draft`, `workItemTeam` from the URL querystring (defaults: org/project `SnelStart`, status `Active`). Invalid values silently fall back to defaults.
2. **PAT gate** — the Azure DevOps Personal Access Token (Code Read scope) is never persisted to disk; it lives only in `sessionStorage` (tab-scoped) under `PAT_STORAGE_KEY`. A PAT can also be passed via `?pat=...`, which is immediately captured into `sessionStorage` and stripped from the URL/history so it doesn't linger in the address bar or browser history.
3. **Azure DevOps API** — talks directly to `dev.azure.com` REST API (v7.1) from the browser using Basic auth (`btoa(':' + pat)`). Three calls per load cycle: paginated PR list, per-PR linked work item IDs (fetched with bounded concurrency via `runWithConcurrency`, limit 10), and a batched work item details lookup (200 IDs/batch, with automatic batch-splitting retry on failure).
4. **Loading orchestration** — `loadPrData` drives the multi-step fetch with progress reporting, then shapes raw Azure DevOps PR objects into the flat row shape used by rendering (`repository`, `id`, `title`, `status`, `createdBy`, `teams`, `approvers`, `tags`, `workItems`, `wiTeams`, `created`, `isDraft`, `url`). Results are cached in `sessionStorage` per `org/project/status` combo (`DATA_CACHE_KEY`) so reloading the page doesn't re-hit the API; "Refresh data" bypasses this cache.
5. **Report rendering** — builds grouped HTML tables (`renderReport`) with click-to-sort columns, five client-side filters (user, team, work item tag, work item team, draft, approval), and reflects all filter/sort/group state into the URL querystring via `history.replaceState` (no page reload) so any view is bookmarkable/shareable. Filter dropdowns dynamically narrow their own options to values still reachable given the other active filters (`narrowSelectOptions`/`matchesAllExcept`), excluding each dropdown's own constraint so it can still be widened.

## Conventions to preserve

- No frameworks, no build tooling, no external dependencies — keep it that way; this is meant to be a drop-in static file.
- PATs must never be written to `localStorage`, cookies, or anywhere persistent — `sessionStorage` only, and always stripped from the URL after capture.
- Cell content in the report table uses a `cell-clip` span wrapper (not relying on `<td>` overflow directly) because CSS ellipsis doesn't reliably apply when a cell contains inline-block "pill" elements rather than a plain text run.
- All dynamic text inserted into HTML goes through the local `esc()` helper (textContent → innerHTML escape) to avoid XSS from Azure DevOps data (titles, names, tags, etc.).
