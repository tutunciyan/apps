---
title: 'Selectable Azure DevOps project on the PAT gate and report'
type: 'feature'
created: '2026-09-23'
status: 'done'
route: 'dispatch'
review_loop_iteration: 0
context: []
baseline_commit: '38197ebdea8dcf55ef69f738913c34f8b558402c'
---

<frozen-after-approval reason="human-owned intent — do not modify unless human renegotiates">

## Intent

**Problem:** The Azure DevOps organization ("SnelStart") has multiple projects, but the app only lets you pick one via manually editing `?project=` in the URL — there is no UI control, so most users never discover it and stay stuck on the `SnelStart` project default.

**Approach:** Add a project selector to the PAT gate, pre-selected to today's default (`SnelStart`, or whatever `?project=` already specifies); once a PAT is typed, fetch the org's live project list (Azure DevOps Core API) and populate it. The same live selector is also shown in the loaded report's top bar, so a user can switch projects and reload the report in place, without re-entering the PAT or leaving the page. Both selectors update `CONFIG.project`, the URL's `?project=`, and reuse the app's existing per-project session cache and load/render flow — no parallel code path.

## Boundaries & Constraints

**Always:**
- Preserve `SnelStart` as the effective default project when the user makes no selection and no `?project=` is present.
- Keep the organization fixed — this change scopes to project selection only, not organization selection.
- Route the selected project through the same `CONFIG.project` used everywhere today (API URLs, cache key, page title, PAT-gate label) — no parallel/duplicate state.
- Render every ADO-sourced string (project names) safely: `esc()` only exists inside `renderReport`'s closure, so code that runs outside it (the PAT-gate blur handler, `startLoad`) must build `<option>`s via `document.createElement`/`textContent` instead — never raw string-concatenated into `innerHTML`.
- Sync the selected project into the URL via `history.replaceState`, following the same `URLSearchParams` + "omit when default" pattern `syncUrlToState()` already uses for filters.
- Switching projects (from either selector) must reuse the existing `startLoad`/`dataCacheKey()` flow — same cache-check, loading pane, and render path already used for the initial load and "Refresh data".
- Stay a static, dependency-free, build-free file set (plain DOM APIs only).

**Never:**
- Do not make the organization selectable.
- Do not persist the PAT or fetched project list anywhere beyond existing `sessionStorage` usage (the fetched project-name list itself may be kept in an in-memory JS variable for the tab session, not persisted storage).
- Do not block or error out the gate if the project-list fetch fails — the PAT's validity is still verified by the existing PR-load call on submit.

## I/O & Edge-Case Matrix

| Scenario | Input / State | Expected Output / Behavior | Error Handling |
|----------|--------------|---------------------------|----------------|
| Default load | No `?project=` in URL, gate freshly rendered | `#projectSelect` shows one option, `SnelStart`, selected; `patOrgProjectLabel` reads `SnelStart/SnelStart` | N/A |
| URL override | `?project=snelstart-internal` in URL | `#projectSelect` pre-selects `snelstart-internal`; label matches | N/A |
| PAT entered, field blurred | Non-empty PAT typed, `#patInput` loses focus | Org project list fetched once (cached in memory for the tab); `#projectSelect` repopulated with real project names, current selection preserved | On failure, dropdown stays unchanged, no error banner; retried on the next blur |
| Report loads (gate submit, cache hit, or fast PAT-already-in-session path) | `startLoad` runs, report about to render | Project list is fetched if not already cached this session, and both `#projectSelect` and the report top bar's `#topBarProjectSelect` are populated/kept in sync with `CONFIG.project` | If the fetch fails, both selectors fall back to a single option (current project) |
| User changes the gate selector | A different project is chosen from the populated `#projectSelect` before PAT submit | `CONFIG.project`, `patOrgProjectLabel`, and `?project=` all update immediately; `?project=` is omitted when the choice equals `SnelStart` | N/A |
| User switches project from the report top bar | Report already loaded, different project chosen in `#topBarProjectSelect` | `CONFIG.project`/URL update the same way, then `startLoad` re-runs for the new project (cache reused if present, otherwise the normal loading pane runs) and the report re-renders in place | If the reload fails, the existing gate error path is shown, same as any other load failure |
| Selected project missing from fetched list | e.g. a typo'd `?project=` or a since-renamed project | The value is kept as an extra dropdown option instead of being silently dropped (same defensive pattern as `narrowSelectOptions`) | N/A |

</frozen-after-approval>

## Code Map

- `pull-requests/index.html:10-19` -- PAT gate markup; insert the `Project` field here, between the intro paragraph and the PAT input.
- `pull-requests/index.html:27-37` -- `#topBar .actions` markup (Refresh data / Forget PAT buttons); add `#topBarProjectSelect` alongside them.
- `pull-requests/main.js:4-27` -- Configuration block; `CONFIG.project` already reads `?project=`. `DATA_CACHE_KEY` is currently a `const` computed once — must become a function since `CONFIG.project` can now change after script load.
- `pull-requests/main.js:29,37-53` -- PAT gate section; `patOrgProjectLabel` is set once at top level — needs to become a re-callable `updatePatGateLabel()`. `capturePatFromUrlIfPresent()` (lines 44-52) is the existing reference pattern for mutating the querystring via `history.replaceState` outside of `renderReport`.
- `pull-requests/main.js:87-110` -- Azure DevOps API section; reuse `apiGet`/`authHeader` for the new project-list call.
- `pull-requests/main.js:330-362` -- `startLoad()`; the single place both the gate-submit path and the fast "PAT already in session" path converge — project-list fetch + both selectors' sync must happen here so it works regardless of entry path or cache hit/miss.
- `pull-requests/main.js:600-618` -- `syncUrlToState()` inside `renderReport`; reference pattern for the `setOrDelete`-style URL sync the new `updateProjectInUrl()` should mirror (but must live at top-level scope, since it runs before/outside `renderReport`).
- `pull-requests/main.css:45-56` -- `#patGate` styling block; extend with a `select` rule matching the existing input look. `pull-requests/main.css:65-68` -- `#topBar` block; extend similarly for `#topBarProjectSelect`.

## Tasks & Acceptance

**Execution:**
- [x] `pull-requests/index.html` -- add `<div class="field"><label for="projectSelect">Project</label><select id="projectSelect"></select></div>` inside `#patGate`, after the intro `<p>` and before the PAT `<label for="patInput">`.
- [x] `pull-requests/index.html` -- add `<select id="topBarProjectSelect"></select>` inside `#topBar .actions`, alongside the existing Refresh/Forget buttons.
- [x] `pull-requests/main.js` -- add `const DEFAULT_PROJECT = 'SnelStart';` immediately before the `CONFIG` object literal, and change `CONFIG.project`'s fallback from the `'SnelStart'` literal to `DEFAULT_PROJECT` (single source of truth for the default); replace `const DATA_CACHE_KEY = ...` with `function dataCacheKey() { return 'azdo-pr-report-cache:' + CONFIG.organization + '/' + CONFIG.project + '/' + CONFIG.status; }`; update its three existing reads (`readCache`, `writeCache`, `forgetPat`) to call `dataCacheKey()`.
- [x] `pull-requests/main.js` -- add `updatePatGateLabel()` (sets `patOrgProjectLabel.textContent` from live `CONFIG`), call it once at startup in place of the old one-line assignment, and again whenever the project changes from either selector.
- [x] `pull-requests/main.js` -- add `updateProjectInUrl(project)`, mirroring `syncUrlToState`'s `setOrDelete`: set `?project=` when it differs from `DEFAULT_PROJECT`, delete it otherwise; use `URLSearchParams` + `history.replaceState` like `capturePatFromUrlIfPresent`.
- [x] `pull-requests/main.js` -- add `async function fetchProjects(pat)`: `GET https://dev.azure.com/{CONFIG.organization}/_apis/projects?api-version=7.1&$top=100` via `apiGet`, return `(response.value || []).map(p => p.name)`.
- [x] `pull-requests/main.js` -- add `let cachedProjectNames = null; let projectListPromise = null; async function ensureProjectList(pat) { ... }`: returns `cachedProjectNames` immediately if already resolved this tab session; otherwise, if a fetch is already in flight (`projectListPromise` set), await and return that same promise instead of starting a second one (avoids the gate's blur handler and `startLoad` racing two concurrent `_apis/projects` calls); otherwise start `fetchProjects(pat)`, store it as `projectListPromise`, and on completion set `cachedProjectNames` to the result (or leave it `null` on failure, clearing `projectListPromise` either way so a later call can retry).
- [x] `pull-requests/main.js` -- add `populateProjectSelect(selectEl, names)`: rebuild `selectEl`'s `<option>`s using `document.createElement('option')` + `textContent` for each name in `names` (sorted) -- not `esc()`/`innerHTML`, since this runs outside `renderReport`'s closure -- keeping the currently selected value selected even if absent from `names` (append it as an extra option, same defensive pattern as `narrowSelectOptions`, main.js:673-677). Used for both `#projectSelect` and `#topBarProjectSelect`.
- [x] `pull-requests/main.js` -- seed `#projectSelect` with a single option equal to `CONFIG.project` (selected) at startup, before any fetch. (Also seeded `#topBarProjectSelect` the same way -- see Implementation Notes.)
- [x] `pull-requests/main.js` -- wire `#patInput` `blur`: if the trimmed PAT is non-empty, call `ensureProjectList(pat)`; if it returns a list, call `populateProjectSelect(projectSelect, names)`.
- [x] `pull-requests/main.js` -- wire `#projectSelect` `change`: set `CONFIG.project = projectSelect.value`, then call `updatePatGateLabel()` and `updateProjectInUrl(CONFIG.project)`.
- [x] `pull-requests/main.js` -- wire `#topBarProjectSelect` `change`: set `CONFIG.project = topBarProjectSelect.value`, call `updatePatGateLabel()` and `updateProjectInUrl(CONFIG.project)`, then call `startLoad(getPat(), { forceRefresh: false })` to reload the report for the new project through the existing cache/loading flow.
- [x] `pull-requests/main.js` -- in `startLoad`, after `prs` is resolved (from cache or freshly fetched) and before/around `renderReport(prs)`: call `ensureProjectList(pat)`; if it returns a list, `populateProjectSelect` both `#projectSelect` and `#topBarProjectSelect` (if present) from it; if it returns `null` (fetch failed), fall back to `populateProjectSelect(selectEl, [CONFIG.project])` for both so neither is ever left with zero options. Either way, set both selects' `.value` to `CONFIG.project` so they stay in sync on every load, refresh, and project switch.
- [x] `pull-requests/main.css` -- add `#patGate select { width:100%; box-sizing:border-box; padding:8px 10px; border:1px solid #c8c8c8; border-radius:4px; font-size:13px; margin-bottom:14px; }` and a matching compact rule for `#topBar select` (reuse `#topBar button`'s padding/border/radius/font-size).

**Acceptance Criteria:**
- Given a fresh page load with no `?project=` and no interaction, when the user submits their PAT, then pull requests load for `SnelStart` exactly as before this change.
- Given the user selects a different project on the gate, when they then click "Refresh data" after the report loads, then the refreshed data stays scoped to that selected project (cache key and API calls both use the updated `CONFIG.project`).
- Given the user bookmarks the URL right after picking a non-default project (before submitting the PAT), when they reopen that bookmark, then the gate pre-selects that same project.
- Given the report is already loaded, when the user picks a different project from the top bar selector, then the report reloads for that project in place (loading pane if uncached, instant if cached) without returning to the PAT gate or re-entering the PAT.
- Given the PAT was already in `sessionStorage` so the gate is skipped entirely on page load, when the report finishes loading, then the top bar's project selector is still correctly populated with the org's real project list.

## Implementation Notes

- `#topBarProjectSelect` is also seeded with a single `[CONFIG.project]` option at startup (beyond the literal task-71 wording, which only named `#projectSelect`). Without it, the top bar select's `.value` starts `''` (no options yet), so the first time `syncProjectSelectors` populates it from the fetched list, there'd be no "current value" to preserve if that project were missing from the list (e.g. a typo'd `?project=`) -- `.value = CONFIG.project` would then silently fail to select anything. Seeding both selectors identically closes this gap and keeps them symmetric.
- Manually verified via a local static server + browser: default gate state (`SnelStart` selected, label matches), `?project=snelstart-internal` URL override pre-selecting correctly, and the PAT-blur project-list fetch -- confirmed exactly one `GET .../_apis/projects?api-version=7.1&$top=100` request fires, and on failure (invalid PAT) the dropdown is left unchanged with no error banner and no console errors, matching the spec's error-handling rows.
- Not verified end-to-end (no real Azure DevOps PAT/org available in this environment): the live report view, the top-bar selector's real population from a genuine project list, switching projects from the top bar and the reload-in-place behavior, and per-project cache isolation on "Refresh data". These are called out in the spec's own Verification section as needing a real PAT/org and should be exercised manually against the live org before/soon after this ships.

## Review Triage Log

Review pass 1 (blind-hunter, edge-case-hunter, verification-gap), against diff since `baseline_commit`.

- **[high, patch]** Concurrent `startLoad` calls race on shared mutable `CONFIG.project` (edge-case-hunter #3/#4; corroborated by verification-gap's cache-isolation gap). Verified: `topBarProjectSelectEl`'s `change` handler fires `startLoad` without awaiting it; `loadPrData`'s inner calls (`fetchLinkedWorkItemIds`, the final PR-mapping, `writeCache`) re-read live `CONFIG.project` after `await` points, so a second rapid project switch before the first load finishes can write one project's PR data into another project's `sessionStorage` cache slot and render it under the wrong label, with no error shown. Real and reachable (loading can take several seconds per the progress UI, leaving ample window for a second switch). Routed patch: add a reentrancy guard around `startLoad` so overlapping calls can't interleave, rather than threading a captured project value through every inner function.
- **[medium, patch]** `#topBarProjectSelect`'s `change` handler calls `startLoad(getPat(), ...)` without checking for a `null` PAT (blind-hunter; same root cause as edge-case-hunter's claims-check #5). Verified: `getPat()` returns `null` whenever "Remember for this browser tab session" was left unchecked; `refreshDataBtn`'s handler already guards this (`if (!pat) { showOnly(patGateEl); return; }`) but the new top-bar handler doesn't, so it sends an unauthenticated request and lands on a confusing "Failed to load... Check that the PAT is valid" message instead of a clean gate redirect — also contradicts the spec's own AC ("without re-entering the PAT"). Routed patch: mirror `refreshDataBtn`'s existing guard.
- **[medium, patch]** PAT-gate instructions still say only "Code (Read)" scope, but `fetchProjects()` hits Core API `_apis/projects`, which needs "Project and Team (Read)" scope (blind-hunter). Verified against ADO's documented PAT scopes: Code scope does not cover the Core API. A PAT created exactly as instructed will silently fail to populate either selector (caught by `ensureProjectList`'s `.catch(() => null)`, no error shown) — degrades gracefully but defeats the feature's purpose for most first-time users. Routed patch: update the on-screen scope wording.
- **[low, patch]** `ensureProjectList`'s cache/in-flight dedupe (`cachedProjectNames`/`projectListPromise`) is keyed only on "already fetched," not on which PAT produced it (blind-hunter + edge-case-hunter #1, same defect). Verified at `main.js:176-193`: a list fetched for one PAT is silently reused for a later, different PAT with no refetch. Narrow trigger (requires editing the PAT after an initial successful fetch), but the fix is small and direct. Routed patch: track `cachedForPat` and invalidate on mismatch.
- **[low, patch]** `'SnelStart'` is a bare literal in `CONFIG.organization`'s fallback but a named `DEFAULT_PROJECT` constant in `CONFIG.project`'s fallback (blind-hunter). Verified — cosmetic/naming inconsistency only, no functional impact. Fix is a direct one-line correction, so not rejected despite low real-world impact. Routed patch.
- **[low, patch]** `#topBarProjectSelect` has no visible label or `aria-label`, only a `title` tooltip (blind-hunter). Verified against `index.html` — every other top-bar control has visible text. Fix is a trivial attribute addition. Routed patch.
- **[low, patch]** The new gate wrapper `<div class="field">` has no effect: the only existing `.field` CSS rule is scoped `.filters .field`, and this div isn't inside `.filters` (blind-hunter). Verified in browser — renders correctly today purely via `#patGate label`/`#patGate select`, so this is dead/misleading markup, not a visual bug. Fix is a trivial deletion. Routed patch.
- **[low, false]** "`updateProjectInUrl` duplicates `syncUrlToState`'s pattern instead of extending it" (blind-hunter). Refuted: `syncUrlToState` is a closure inside `renderReport` reading multiple `renderReport`-local filter-element `.value`s that don't exist before the report loads; `updateProjectInUrl` must run standalone at gate-time, before `renderReport` ever executes — the spec's own Code Map already documents this as a deliberate necessity, not an oversight. Rejected.
- **[low, reject]** `fetchProjects` hardcodes `$top=100` with no pagination (blind-hunter + edge-case-hunter #2). Verified true, but the org this app targets has ~3 projects (per the feature's own intent) — meeting this in everyday use is implausible, and the fix (a `$skip`-loop like `fetchAllPullRequests`) is more than a direct correction. Rejected per the low-finding rule (both conditions met).
- **[low, reject]** Neither project selector shows a loading/disabled state while the list fetch is in flight (blind-hunter). Verified true, but the fetch is fast (~3 projects) and the dropdown transparently repopulates once ready; the fix requires new UI/loading-state complexity, not a direct correction. Rejected per the low-finding rule (both conditions met).

## Verification

**Manual checks (if no CLI):**
- Serve the repo root with a static file server (per AGENTS.md — `file://` behaves inconsistently with `sessionStorage`/fetch) and open `pull-requests/index.html`.
- Confirm the gate shows `Project: SnelStart` selected by default, and the intro label reads `SnelStart/SnelStart`.
- Type a valid PAT, tab out of the field, and confirm the dropdown repopulates with the org's real projects shortly after (Network tab shows one `GET .../_apis/projects` call).
- Change the gate's selection, confirm the URL's `?project=` updates live and the intro label updates, then submit and confirm the report loads that project's PRs (page title and `metaLine` reflect it) and the top bar selector shows the same project selected.
- From the loaded report, switch the top bar selector to another project and confirm the report reloads in place for that project, the URL updates, and switching back to an already-visited project is instant (served from cache).
- Reload with `?project=snelstart-internal` in the URL and confirm the gate pre-selects it before any PAT is typed.
- Reload with a PAT already remembered in `sessionStorage` (gate skipped) and confirm the top bar selector still populates correctly.
- Confirm "Refresh data" and "Forget PAT" still work correctly against whichever project is currently selected.
