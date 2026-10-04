# Project Pulse Dashboard — Implementation Plan

## 1. Summary

Mona's team needs a small, static **Project Pulse** dashboard so contributors can see, at a glance, which projects are active, who owns them, their status, recent activity, and priority/risk. The dashboard is a static app with no build step or backend: `app/index.html` (structure), `app/styles.css` (visual design), and `app/project-data.json` (mock project data). A `.vscode/launch.json` must let a learner start a local static server rooted at `app/` and auto-open `index.html` in the browser via VS Code's **Run and Debug** panel, using the launch configuration name **"Run Project Pulse Dashboard"**.

Today `app/` and `.vscode/launch.json` do not exist (`.vscode/` only has `tasks.json`). This is greenfield work. The Orchestrator should route visual/structural/accessibility work to **Designer** and code/data/tooling work to **Coder**, per `.github/agents/designer.agent.md` and `.github/agents/coder.agent.md`. Because `app/index.html`, `app/styles.css`, and `app/project-data.json` are logically coupled (markup consumes data and styling hooks), work must be sequenced carefully even though Designer and Coder touch different files, to avoid one agent guessing at contracts (CSS class names, JSON schema) the other hasn't defined yet.

This plan is intentionally prescriptive about the JSON schema, CSS hooks, and launch configuration shape so that Designer and Coder can work with a shared contract and so the result is deterministic and testable (matching repository conventions already encoded in `.github/steps/3-step.md` and `scripts/validate-exercise.sh`).

## 2. Ordered Implementation Steps

**Step 0 — Confirm shared contract (Planner/Orchestrator, no files written)**
Lock in the data schema and CSS hook names before any file is created, since both Designer and Coder depend on them:

- Data schema (top-level `projects` array in `app/project-data.json`), each project object with:
  - `name` (string, required)
  - `owner` (string, required)
  - `status` (string enum-like: e.g. `"On Track"`, `"At Risk"`, `"Blocked"`, `"Completed"` — free text is acceptable but should be used consistently in sample data)
  - `recentActivity` (string, short human-readable sentence, e.g. "Merged PR #42 — updated onboarding flow")
  - `priority` (string enum-like: `"High"`, `"Medium"`, `"Low"`)
- CSS/markup hooks: `.dashboard` (outer container), `.project-card` (per-project card), plus implied sub-hooks for status/priority treatment (e.g. `.status-badge`, `.priority-high/.priority-medium/.priority-low` or `data-status`/`data-priority` attributes — Designer's call, but must be consistent with what Coder renders).
- Launch contract: configuration name **"Run Project Pulse Dashboard"**, `cwd` = `${workspaceFolder}/app`, command `python3 -m http.server 5500`, `serverReadyAction` pattern opening `http://localhost:%s/index.html`.

**Step 1 — Coder: Author `app/project-data.json`**
Create the mock data file first since markup and styling both consume it. Include a representative, varied sample (3–6 projects) covering every status and priority value so Designer/Coder can visually verify badge/priority treatment and edge cases (long name, missing/short recentActivity, etc.) without guessing.

**Step 2 — Coder: Author `app/index.html`**
Build semantic HTML: a page title "Project Pulse", a `<div class="dashboard">` (or `<main class="dashboard">`) container, and a mechanism to render `.project-card` elements from `project-data.json` (inline `<script>` that `fetch()`s the JSON and renders cards, since this is a static app with no build tooling/bundler and no server-side rendering). Link `styles.css` via `<link rel="stylesheet">`. Each rendered card must surface `name`, `owner`, `status`, `recentActivity`, and `priority`. Use deterministic class names (`project-card`, plus hooks for status/priority) so Designer's CSS has stable targets. Include basic accessible structure (heading levels, landmark roles) even though fine-grained accessibility polish is Designer's responsibility.

**Step 3 — Designer: Author `app/styles.css`**
Style the `.dashboard` container (responsive grid/flex layout) and `.project-card` (rounded corners, shadow, spacing, hover/focus states), add visually distinct status badges and priority treatment (color + icon/text, not color alone, for accessibility), ensure responsive behavior at narrow widths (stack cards, readable type scale), and verify contrast ratios meet WCAG AA.

**Step 4 — Coder: Author `.vscode/launch.json`**
Create a strict-JSON (no comments) VS Code launch configuration named **"Run Project Pulse Dashboard"** that serves `app/` on a fixed port (5500) via `python3 -m http.server 5500` with `cwd` set to `${workspaceFolder}/app`, and a `serverReadyAction` that opens `http://localhost:5500/index.html` in the browser (pattern `http://localhost:%s/index.html` keyed off the port). This depends on `app/index.html` existing at the expected relative path (`index.html`, not `app/index.html`, since `cwd` is already `app/`).

**Step 5 — Joint validation pass (Designer + Coder + Orchestrator)**
Confirm the three app files integrate correctly (fetch path, class names, data shape) and that the launch configuration actually opens the dashboard, not a directory listing.

## 3. File Assignments

| File | Owner | Rationale |
|---|---|---|
| `app/project-data.json` | **Coder** | Mock/sample data generation and JSON schema are code/data concerns per Coder's "implements logic" and "deterministic and testable" principles. |
| `app/index.html` | **Coder** | Markup/structure, data-fetch logic, and rendering logic are code concerns. Coder may consult Designer's hook names before finalizing class names, but authors the file. |
| `app/styles.css` | **Designer** | Visual design, card layout, status badges, priority treatment, responsive layout, and accessibility are explicitly Designer's domain per `designer.agent.md`'s "Project Pulse design expectations." |
| `.vscode/launch.json` | **Coder** | Support/tooling configuration for running the app is explicitly called out in `coder.agent.md`'s "Runnable app support" section — Coder owns this even though it's not application code. |

Note: Designer must **not** touch `app/index.html` or `.vscode/launch.json`; Coder must **not** touch `app/styles.css` (per each agent's "stay within assigned files" / "do not change design-only files unless explicitly assigned" rules). If Coder needs a new CSS hook mid-implementation, that should be raised back to the Orchestrator/Designer rather than edited directly.

## 4. Dependencies Between Steps

- **Schema before markup**: `app/project-data.json`'s field names (`name`, `owner`, `status`, `recentActivity`, `priority`) must be finalized before `app/index.html`'s rendering script is written, since the script's property accesses hard-depend on exact key names.
- **Markup before styling**: `app/styles.css` needs `app/index.html` to exist first (or at minimum, the agreed class-name contract) so Designer can target real selectors (`.dashboard`, `.project-card`, badge/priority hooks) rather than speculative ones.
- **Data before styling (soft dependency)**: Designer benefits from seeing representative data (varied status/priority values, long names) to style badges and handle overflow/wrapping correctly.
- **`app/index.html` before `.vscode/launch.json`**: the launch config's `serverReadyAction` opens `index.html` at a specific relative path; the file should exist (or at least its filename/location be fixed) before finalizing the launch config, though in practice the launch config's *shape* doesn't require the file's *content* to be done — just its path.
- **All three app files before final validation**: full integration testing (open the server, confirm cards render with correct data and styling) requires all of `index.html`, `styles.css`, and `project-data.json` to be in place.

## 5. Parallel vs. Sequential Work

**Must run sequentially:**
- `app/project-data.json` → `app/index.html`: hard data dependency (property names consumed by rendering logic).
- `app/index.html` (class names decided) → `app/styles.css`: Designer needs stable selectors to style against; styling a non-existent or still-changing DOM structure risks rework.

**Can run in parallel:**
- `.vscode/launch.json` can be authored at any point *after* the Step 0 contract is agreed (specifically, after everyone agrees `index.html` lives at `app/index.html` and the server should run from `app/`). It has no file-content dependency on `index.html`/`styles.css`/`project-data.json` — only a path/filename dependency, which is fixed up front. This can proceed in parallel with Step 1–3 since it's a different file with no overlapping scope.
- Within Designer's own work, exploring/polishing the responsive breakpoints and badge color system can happen in parallel with Coder writing `launch.json`, since file scopes don't overlap (`app/styles.css` vs. `.vscode/launch.json`).

**Why this split**: the Orchestrator's execution model explicitly says "run tasks in parallel only when file scopes do not overlap and there are no data dependencies." `project-data.json` → `index.html` → `styles.css` is a real data/selector dependency chain and must be sequential. `launch.json` has no such dependency once the `app/index.html` path is agreed, so it's the one piece safely parallelizable with the HTML/CSS work.

## 6. Edge Cases to Handle

- **Empty `projects` array**: `index.html`'s rendering script should show a friendly "No projects yet" state in `.dashboard` rather than a blank page or JS error.
- **Missing/undefined fields**: if a project object is missing `recentActivity` or `priority`, the renderer should degrade gracefully (e.g., show "—" or omit the badge) instead of rendering `undefined` as text or throwing.
- **Long project names / long `recentActivity` text**: CSS must wrap/truncate gracefully (e.g., `overflow-wrap: break-word`, optional `text-overflow: ellipsis` with care for accessibility) rather than breaking card layout.
- **Large number of projects** (e.g., 20+): grid/flex layout should wrap predictably without unbounded horizontal scroll; consider a max-width container.
- **Unknown/unexpected `status` or `priority` values**: styling should have a sensible default badge style for any value not explicitly matched, rather than rendering unstyled or broken.
- **Narrow/mobile viewport**: cards must stack in a single column with readable text size and tappable spacing; verify no horizontal overflow.
- **JSON fetch failure** (e.g., opening `index.html` via `file://` instead of through the launch server): `fetch()` of a local JSON file is blocked by browser CORS policy under `file://`. The dashboard **must** be run through the `.vscode/launch.json` HTTP server for `fetch()` to succeed — this should be documented/expected behavior, and ideally the HTML includes a visible fallback/error message if the fetch fails, so learners don't see a silently blank dashboard when double-clicking the HTML file directly.
- **Port conflicts**: port 5500 may already be in use in a Codespace; the launch config should use a single deterministic port as specified, and if conflicts occur that's a known limitation to document rather than solve automatically (VS Code's `serverReadyAction` will still attempt to open the specified port).
- **Case sensitivity / typos in status and priority strings**: sample data and CSS selectors/attribute matchers should agree on exact casing (e.g., `"At Risk"` vs `"at-risk"`) to avoid silently unstyled badges.
- **Accessibility**: status/priority must not be conveyed by color alone (colorblind users) — include text labels or icons alongside color.

## 7. Validation Expectations

**Coder should verify:**
- `app/project-data.json` parses as valid JSON (`python3 -m json.tool app/project-data.json`) and has a top-level `projects` key whose entries each include `name`, `owner`, `status`, `recentActivity`, `priority`.
- `app/index.html` references `styles.css` and loads/fetches `project-data.json` (open the file through the launch server, not `file://`, and confirm cards render with correct values for each sample project).
- Every rendered card uses the `project-card` class, and `status`, `recentActivity`, and `priority` values are visibly present in the rendered output.
- `.vscode/launch.json` parses as strict JSON (`python3 -m json.tool .vscode/launch.json`, no trailing commas or comments).
- `.vscode/launch.json` contains a configuration literally named `"Run Project Pulse Dashboard"`, with `cwd` set to `${workspaceFolder}/app`, and a `serverReadyAction` pattern that opens `http://localhost:%s/index.html`.
- Manually run **Run and Debug → Run Project Pulse Dashboard** in VS Code/Codespaces and confirm the browser opens the actual Project Pulse dashboard UI (not a directory index listing) at `http://localhost:5500/index.html`; stop the server afterward.

**Designer should verify:**
- `.dashboard` and `.project-card` selectors exist in `app/styles.css` and are visibly applied when the page renders through the launch server.
- Cards show rounded corners (`border-radius`) and shadow (`box-shadow`), with clear visual separation and readable spacing.
- Status badges and priority indicators are visually distinct and not conveyed by color alone.
- Resize the browser (or use responsive design mode) to confirm cards reflow/stack sensibly at narrow widths (e.g., ~375px) without horizontal scrolling or overlapping text.
- Spot-check contrast ratios for text-on-badge and text-on-card combinations against WCAG AA (4.5:1 for normal text).
- Confirm long names/activity text don't break card layout (test against the deliberately-varied sample data from Step 1).

**Joint/Orchestrator validation:**
- End-to-end: open the dashboard via the launch configuration and confirm all sample projects render correctly with matching styling and no console errors (check browser dev tools for fetch/JS errors).
- Re-run the repository's own exercise validation script (`scripts/validate-exercise.sh`) if available in context, since it already encodes many of these exact checks (JSON validity, required selectors, required keyphrases, launch config name) and will catch regressions mechanically.

## 8. Open Questions

1. **Status/priority vocabulary**: should `status` and `priority` be strictly constrained to an enum (e.g., only `"On Track" | "At Risk" | "Blocked" | "Completed"`), or should the schema stay as loosely-typed strings with "typical" sample values? This plan assumes loose strings with consistent sample usage, but Designer's CSS selector strategy (class-based vs. `data-*` attribute-based) depends on this being nailed down early.
2. **Rendering approach**: should `index.html` use an inline `<script>` with `fetch()`, or would a `<script src="app.js">` be preferred for separation of concerns? The brief only lists three app files (`index.html`, `styles.css`, `project-data.json`), implying inline script is expected — confirm before Coder starts, since adding a fourth JS file isn't in the brief's explicit file list.
3. **Port 5500 availability in Codespaces**: is this port already used by another forwarded service in this environment? If so, the deterministic port requirement from `.github/steps/3-step.md` may need a documented exception.
4. **Icon usage**: should priority/status badges use simple text/emoji (no external dependencies, keeps the app fully static/offline) or is an icon font/SVG sprite acceptable? Given "static app" and no build step, plain text/emoji is recommended to avoid introducing external asset dependencies.
5. **Number of sample projects**: brief doesn't specify exact count — this plan recommends 3–6 to cover edge cases (varied status/priority, one long name) without cluttering the first-run demo; confirm this range is acceptable to Mona/Orchestrator.
