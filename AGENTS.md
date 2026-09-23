<!-- bmad:context -->
<!-- Verified 2026-09-23 against 144156f. Managed by bmad-project-context; edits inside this block are replaced on refresh. Keep anything you want preserved outside the markers. -->

## apps — Azure DevOps Pull Requests report

Static, client-side-only dashboard that reads PRs and linked work items from the Azure DevOps REST API (v7.1) straight from the browser. Plain HTML/CSS/JS: no server, no package manager, no build, no tests. BMad planning output goes to `_bmad-output/`.

## Policy

- Commit directly to `main`; no branch or PR needed.
- No frameworks, build tooling, or external dependencies — it must stay a drop-in static file set. Use plain DOM APIs instead.
- Never write the PAT to `localStorage`, cookies, or anywhere persistent; `sessionStorage` only, and strip `?pat=` from the URL and history immediately after capture.
- Pass all Azure DevOps data (titles, names, tags, …) through the local `esc()` helper before inserting it into HTML.
- Keep `PullRequests.html` redirecting into `pull-requests/index.html` with query string and hash forwarded — old bookmarks depend on it. Update it when moving or renaming the app.
- Never hand-edit `_bmad/`, `.agents/skills/`, `.github/agents/`, or `.claude/skills/` — installer-managed. Put overrides in `_bmad/custom/`.

## Where things are

- App: `pull-requests/index.html` (shell), `pull-requests/main.js` (all logic, one IIFE), `pull-requests/main.css`.
- Navigate `main.js` by its `// ==== <Section> ====` banners: Configuration, PAT gate, Azure DevOps API, Loading orchestration, Report rendering, Entry point.
- Exclude `.agents/`, `_bmad/`, and `.github/agents/` from code searches — BMad tooling, ~97% of tracked files, unrelated to the app.

## Running and verifying

- Open `pull-requests/index.html` in a browser; serve the repo root with a static file server instead when testing `?pat=` or PAT/cache persistence — `file://` behaves inconsistently with `sessionStorage` and fetch.
- No automated tests or linters exist; verify every change manually in a browser, including after "Refresh data" (re-renders without a page reload).

## Conventions that differ from defaults

- Wrap report cell content in a `cell-clip` span; don't rely on `<td>` overflow — ellipsis fails on cells holding inline-block pills.
- All filter, sort, and group state lives in the querystring via `history.replaceState` (never a reload). A new filter needs all of: a `CONFIG` entry with silent fallback for invalid values, `readInitialStateFromUrl()`, `syncUrlToState()`, a `filterMatchers` entry, and dropdown narrowing via `narrowSelectOptions`/`matchesAllExcept`.
- PR data is cached in `sessionStorage` per `org/project/status` (`DATA_CACHE_KEY`); only "Refresh data" bypasses it, so a changed data shape needs a cache-busting key change or a manual clear.

## Known pitfalls

- Read filter state from the live `window.location.search`, never the page-load `qs`/`CONFIG` snapshot — the snapshot reset users' filters after "Refresh data" (fixed in `038fc2a`).

<!-- /bmad:context -->
