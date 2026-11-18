# Tasks: GitHub Notes Viewer

**Input**: Design documents from `/specs/001-github-notes-viewer/`
**Branch**: `001-github-notes-viewer`
**Date**: 2026-11-18

**Prerequisites**: [plan.md](./plan.md), [spec.md](./spec.md), [research.md](./research.md), [data-model.md](./data-model.md), [contracts/](./contracts/)

**Tests**: Manual validation only per spec — see quickstart.md scenarios. No automated test harness planned (consistent with Principle I).

---

## Phase 1: Setup (Shared Infrastructure)

**Purpose**: Project scaffolding under `apps/github-notes-viewer/`.

- [ ] T001 Create directory structure per plan.md project layout: `apps/github-notes-viewer/{css/,js/components/,assets/}`, `workers/`, root config files
- [ ] T002 [P] Initialize `apps/github-notes-viewer/package.json` with empty `devDependencies` (wrangler CLI optional; keep it minimal per Principle I)
- [ ] T003 [P] Create `index.html` shell with semantic two-pane `<main>` layout: `<aside id="tree-pane">` (left) and `<section id="reader-pane">` (right), plus a `<header>` for repo input and auth status
- [ ] T004 [P] Add CDN script tags in `<head>` for `marked`, `DOMPurify`, `Prism` (core + common languages), `vanillatree` (ARIA tree widget), and `gray-matter`; defer execution until DOM ready
- [ ] T005 [P] Create `css/main.css` with CSS custom properties for theming (WCAG AA contrast minimums per Principle VIII), two-pane flexbox layout, responsive breakpoints for mobile (< 768px collapses to single pane with tree toggle)

**Checkpoint**: Static page loads with layout structure but no interactivity yet.

---

## Phase 2: Foundational (Blocking Prerequisites)

**Purpose**: Auth flow, API client, and URL routing that every user story depends on. Cannot begin US work until complete.

- [ ] T006 Implement Cloudflare Worker proxy skeleton in `workers/proxy.ts`: middleware that reads `GITHUB_CLIENT_ID` / `GITHUB_CLIENT_SECRET` from environment, forwards authenticated requests to `https://api.github.com` with the token from the HttpOnly cookie, returns unauthenticated errors as JSON
- [ ] T007 Configure `wrangler.toml` with Workers binding for environment variables (`GITHUB_CLIENT_ID`, `GITHUB_CLIENT_SECRET`), Pages deployment target, and route rules so `/auth/*` hits the worker while static assets go to Pages
- [ ] T008 Implement OAuth redirect handler in `js/auth.js`: detect PKCE code exchange callback at `/auth/callback`, extract `access_token` from GitHub's response, set an `HttpOnly, Secure, SameSite=Lax` cookie via a worker endpoint (`POST /auth/token`), clear URL search params; expose ` isAuthenticated()` and `getRepoInput()` helpers
- [ ] T009 Implement `js/api.js` with three primitives: `fetchTree(owner, repo)` calling `GET /repos/{owner}/{repo}/git/trees/{sha}?recursive=1` (D-001); `fetchNote(owner, repo, path, etag?)` calling `GET /repos/{owner}/{repo}/contents/{path}` with conditional `If-None-Match` when etag is provided (D-002); `fetchRepoMeta(owner, repo)` calling `GET /repos/{owner}/{repo}` to resolve `default_branch` once per session (D-003). All calls routed through the Worker proxy so the secret never reaches the browser. Return typed objects matching `contracts/github-tree.json` and `contracts/github-note.json`.
- [ ] T10 Implement client-side router in `js/app.js`: parse URL pattern `/{owner}/{repo}/{path?}` via `window.location.pathname`; emit `repoSelected(owner, repo)` and `noteOpened(path)` events; pushState on navigation so bookmarks persist (FR-007b); initial load re-hydrates from URL if present
- [ ] T11 Implement error handling middleware in `js/app.js`: catch 404 / 403 from `api.js` and surface FR-008 friendly messages in the reading pane; implement retry button that increments `retryCount` and refetches; log `X-RateLimit-Remaining` to console during dev per D-002

**Checkpoint**: With a valid GitHub OAuth App configured and deployed, navigating to any `/{owner}/{repo}` URL triggers the auth check and either redirects to GitHub or proceeds to the main UI. No tree or note rendering yet — just auth + routing + API plumbing working end-to-end.

---

## Phase 3: User Story 1 — List all notes from a GitHub repo (Priority: P1) 🎯 MVP

**Goal**: FR-001 + FR-002 + FR-003. User enters a repo name or URL; app fetches the recursive tree and renders every `.md` file as a leaf node in the left pane within ~3 seconds.

**Independent Test**: Point the app at a GitHub repo containing markdown notes; verify every note appears in the tree view (quickstart.md Scenario US-1).

### Implementation for User Story 1

- [ ] T012 [US1] Wire repo input into the header: accept `owner/repo` or full `https://github.com/owner/repo` URL (FR-001), normalize to `{owner, name}`, call `fetchRepoMeta` then `fetchTree` with the resolved `default_branch` SHA, emit `repoLoaded(repoMeta, treeNodes)` on success
- [ ] T013 [P] [US1] Build `TreeNode[]` from the raw GitHub trees API response in `js/app.js` per construction rules in `contracts/github-tree.json`: sort lexicographically by path, prune directories with no `.md` descendants, omit non-.md blobs, derive stable `id` from normalized path, tag `isFolder` boolean
- [ ] T014 [US1] Render the tree nodes into the `<aside id="tree-pane">` using `vanillatree` (D-006) initialized with `role="tree"`, `aria-level`, `aria-expanded`, and `aria-selected` attributes per WAI-ARIA 1.2 treeview pattern (Plan Constitution Check Principle VIII). Emit `nodeClicked(node)` event on selection.
- [ ] T015 [US1] Connect `nodeClicked` to the router: pushState to `/{owner}/{repo}/{path}` (FR-007b), trigger `noteOpened(path)` which sets `ReadingViewState.activeNotePath` but does not yet render content (US2 handles that)
- [ ] T016 [US1] Handle empty-state branch: if `treeNodes.length === 0` after successful fetch, display "No notes found in this repo." in the tree pane instead of an empty list (Edge Cases section of spec.md)
- [ ] T017 [US1] Display loading spinner in the tree pane during the `fetchTree` call; hide on completion or transition to error state per T11

**Checkpoint**: US1 fully functional — enter a repo, see all `.md` files listed in the tree, URL updates on click but nothing renders in the right pane yet. Validate against quickstart.md Scenario US-1.

---

## Phase 4: User Story 2 — Read a note's full content (Priority: P2)

**Goal**: FR-004 + FR-005. Clicking a note in the tree renders its markdown with headings, lists, syntax-highlighted code blocks, and sanitized HTML in the reading pane.

**Independent Test**: Click a note in the tree view; verify its full content renders in the reading view (spec.md acceptance scenarios 1 & 2).

### Implementation for User Story 2

- [ ] T018 [US2] In `js/app.js`, listen for `noteOpened(path)`; when fired, call `fetchNote(owner, repo, path, cachedEtag)` where `cachedEtag` comes from the in-memory `Note` map keyed by `(owner, repo, path)` if one exists (D-002 ETag logic)
- [ ] T019 [US2] On 200 response: decode base64 `content` field from the GitHub response, split front matter with `gray-matter` (D-007), extract `title` from front matter or fall back to filename stem, store the parsed object as a `Note` entity in memory keyed by path, update `ReadingViewState.note` and `isLoading = false`
- [ ] T020 [US2] On 304 response: preserve existing `Note.content` unchanged, update only `fetchedAt` timestamp, set `isLoading = false` without re-rendering (D-002 conditional-request flow)
- [ ] T021 [US2] Render the note in `js/components/reader.js`: pipe `note.content` through `marked.parse()`, sanitize via `DOMPurify.sanitize()`, insert into the reading pane, then call `Prism.highlightAll()` to re-highlight any `<pre><code>` blocks (contracts/github-note.json rendering pipeline)
- [ ] T022 [US2] Populate reading view header with `note.title` and `note.lastModified` timestamp; show front matter metadata invisibly (per Data & File Conventions — never rendered as note content)
- [ ] T023 [US2] Handle the zero-byte / empty-content case: if `note.size === 0` or `note.content.trim() === ''`, render an empty-state message in the reading pane rather than a blank panel (Edge Cases)
- [ ] T024 [US2] Apply CSS styles in `css/main.css` for rendered markdown: readable max-width for prose (~65ch), proper heading hierarchy preservation, code block styling consistent with Prism theme, link `target="_blank" rel="noreferrer noopener"` for external links (DOMPurify hook or post-sanitize traversal)

**Checkpoint**: US1 + US2 together form the core reading experience — tree lists notes, clicking one renders its content. Validate against quickstart.md Scenario US-2 and the US-2 edge cases.

---

## Phase 5: User Story 3 — Browse notes and folders in a tree view (Priority: P2)

**Goal**: FR-006 + FR-007 + FR-007b. Tree shows folder hierarchy, supports keyboard navigation, expand/collapse, and URL bookmarking. This phase completes US1 by adding interactivity and accessibility polish.

**Independent Test**: Point the app at a repo with notes in subdirectories; verify the left panel shows folders and notes inside them, and clicking a note opens its content (spec.md US-3 acceptance scenarios).

### Implementation for User Story 3

- [ ] T025 [US3] Enhance the tree widget with expand/collapse behavior: clicking a folder node toggles `node.isExpanded`, recursively renders or hides children, updates `aria-expanded` on the parent `<li>` (WAI-ARIA treeview pattern). Default expansion mirrors the active note's ancestor path so the tree scrolls into view on navigation.
- [ ] T026 [US3] Implement keyboard navigation per Principle VIII: Tab moves focus into the tree; Arrow Down/Up traverses siblings; Arrow Right expands a collapsed folder; Arrow Left collapses an expanded folder; Enter/Space activates the focused node. Ensure roving tabindex pattern so only one node has `tabindex="0"` at a time.
- [ ] T027 [US3] Scroll the active tree node into view after navigation (programmatic `scrollIntoView({ block: 'nearest' })`) so the user can see where they are in deep hierarchies.
- [ ] T028 [US3] Verify empty-folder pruning matches the construction rule in `contracts/github-tree.json`: directories with no `.md` descendants are omitted entirely, while intermediate directories on the path to a `.md` file remain visible. Test with a repo structure like `notes/projects/alpha.md` where `notes/` and `projects/` are shown but sibling empty folders are not.
- [ ] T029 [US3] Bookmarkability polish: on page load, parse the URL to extract owner, repo, and optional path; if path is present, auto-select the corresponding tree node and trigger note rendering (connects T015 path-forwarding to T018 note-fetching). Validate that reloading a copied URL lands directly on the expected note (quickstart.md Scenario US-3 step 5).

**Checkpoint**: US1 + US2 + US3 together deliver the complete feature. Validate against quickstart.md Scenario US-3 including keyboard and bookmark tests.

---

## Phase 6: Polish & Cross-Cutting Concerns

**Purpose**: Quality gates documented in the quickstart, accessibility audit, deployment readiness.

- [ ] T030 [P] Write `README.md` at repo root covering: how to deploy to Cloudflare Pages + Workers, required environment variables (`GITHUB_CLIENT_ID`, `GITHUB_CLIENT_SECRET`), local dev setup (`npx wrangler pages dev`), known gotchas (HTTPS requirement for HttpOnly cookies during local testing, browser CORS preflight behavior)
- [ ] T031 [P] Add a favicon in `assets/favicon.svg` and an optional `manifest.json` for PWA-install hints (not required for launch but trivial to add); update `<title>` in `index.html` dynamically based on the active repo/note
- [ ] T032 Run the full quickstart.md validation guide against a live deploy or local HTTPS proxy; record sign-off in a new comment block at the bottom of quickstart.md once all six scenarios pass
- [ ] T033 Accessibility audit: run axe-core or equivalent headless audit on the deployed page; verify no WCAG AA violations in the tree widget, reading pane, or error states; fix any flagged issues (keyboard trap, missing labels, insufficient contrast)
- [ ] T034 Rate-limit sanity check: open 20 distinct notes rapidly with DevTools Network tab open; confirm ETag conditional requests return 304 on repeats and that `X-RateLimit-Remaining` stays above a safe threshold (> 100) throughout
- [ ] T035 Bundle-size verification: confirm total gzipped CDN payload (marked + DOMPurify + Prism core + vanillatree + gray-matter) stays under the 50 KB budget stated in research D-006; if over, trim unused Prism languages

---

## Dependencies & Execution Order

### Phase Dependencies

- **Phase 1 (Setup)**: No dependencies — start immediately.
- **Phase 2 (Foundational)**: Depends on Phase 1 — blocks all user stories. Auth + API client + router must exist before any UI interaction works.
- **Phase 3 (US1)**: Depends on Phase 2 — first independently shippable increment (MVP).
- **Phase 4 (US2)**: Depends on Phase 2, integrates with Phase 3 (reads from the tree the user already sees).
- **Phase 5 (US3)**: Depends on Phase 2 and Phase 3 (enhances the tree built there); integrates with Phase 4 (navigation triggers note rendering).
- **Phase 6 (Polish)**: Depends on Phases 3–5 being complete.

### Parallel Opportunities

- **Phase 1**: T002, T003, T004, T005 are fully independent — write simultaneously.
- **Phase 2**: T006 (Worker), T007 (wrangler.toml), T008–T11 (client-side) have a loose dependency — Worker must exist before auth.js can talk to it, but T009–T11 can begin drafting against the expected API shape from contracts/ before the Worker is complete.
- **Phase 3**: T013 (TreeNode builder) is independent of T012 and T014; can be drafted in parallel.
- **Phase 6**: T030 (README) and T031 (favicon) are independent; T032–T035 run sequentially after deployment is live.

---

## Implementation Strategy

### MVP First (User Stories 1 only)

1. Complete Phase 1: Setup
2. Complete Phase 2: Foundational (CRITICAL — blocks all stories)
3. Complete Phase 3: User Story 1
4. **STOP and VALIDATE**: Run quickstart.md Scenario US-1 against a real repo
5. Deploy and demo if ready

### Incremental Delivery

1. Complete Setup + Foundational → foundation ready
2. Add User Story 1 → Test independently → Deploy/Demo (MVP! users can browse note listings)
3. Add User Story 2 → Test independently → Deploy/Demo (users can now read notes)
4. Add User Story 3 → Test independently → Deploy/Demo (complete feature with tree navigation and bookmarks)
5. Each story adds value without breaking previous stories

### Notes

- `[P]` tasks = different files, no cross-task dependencies within the same phase
- `[Story]` label maps the task to a specific user story for traceability back to spec.md
- Each user story should be independently completable and testable — commit after each phase checkpoint
- The `apps/github-notes-viewer/` directory is isolated; no monorepo tooling is required
- Per Principle I, do not add linting/formatting/husky/prettier/etc. until a second contributor justifies it (plan.md Structure Decision)
- Per Principle V (Deliberate Growth), do not add search/filter even though US-4 mentions it — that story is explicitly out of scope
