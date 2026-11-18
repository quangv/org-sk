# Quickstart Validation Guide: GitHub Notes Viewer

**Spec**: [spec.md](./spec.md) | **Plan**: [plan.md](./plan.md) | **Data Model**: [data-model.md](./data-model.md)
**Contracts**: [github-tree.json](./contracts/github-tree.json), [github-note.json](./contracts/github-note.json)
**Date**: 2026-11-18

Manual end-to-end scenarios to prove each User Story before committing implementation. Run these against a real GitHub OAuth App setup and a test repo you control. The scenarios assume `apps/github-notes-viewer/index.html` is served over HTTPS (required for cookie security) either locally via a simple HTTP server or deployed to Cloudflare Pages.

---

## Prerequisites

1. **GitHub OAuth App registered** with:
   - Application name: `org-sk notes viewer (test)`
   - Homepage URL: your deploy preview URL (e.g. `https://notes-viewer.pages.dev`)
   - Authorization callback URL: `<your_deploy_url>/auth/callback`
   - Scopes: `repo` (required for private repos per research D-005)
   - `CLIENT_ID` and `CLIENT_SECRET` saved as Cloudflare Workers environment variables (`wrangler.toml` or dashboard).
2. **Test repository** created under your GitHub account containing:
   ```
   notes/
   ├── index.md
   ├── design.md
   ├── projects/
   │   ├── alpha.md
   │   └── beta.md
   └── archive/
       └── 2024.md
   ```
   At least one file should have YAML front matter including a `title` key so the title-fallback logic can be verified.
3. A local dev server if testing without Cloudflare deploy yet: `npx serve apps/github-notes-viewer` or equivalent. HTTPS is preferred; for local development `http://localhost` is acceptable while auth flows are being wired.
4. Modern browser with JavaScript enabled (Chrome/Firefox/Safari latest).

---

## Scenario US-1: List all notes from a repo

**Goal**: Prove FR-001 + FR-002 + FR-003. Every `.md` file appears in the tree view within ~3 seconds of entering the repo name.

### Steps

1. Open the app URL in the browser.
2. If presented with a login screen, click **Sign in with GitHub**. Complete the OAuth PKCE flow (research D-005). You should land back at the app with an HttpOnly cookie set — verify by opening DevTools → Application → Cookies and confirming a cookie named something like `gh_token` exists with `HttpOnly` checked.
3. In the repo input field, enter `your-username/<your-test-repo>` or the full `https://github.com/your-username/<your-test-repo>` URL. Submit.
4. Observe the left pane populate.

### Expected outcomes

- All four `.md` files appear as leaf nodes under their parent folders (`design.md` under `notes/`, `alpha.md` and `beta.md` under `notes/projects/`, etc.).
- No network errors in the Console.
- Page title reflects the repo name if a description exists on the repo.
- Loading spinner disappears and the tree is interactive within 3 seconds on a typical broadband connection.

### Fail signals

- Any `.md` file missing from the tree.
- `truncated: true` warning in console (indicates the trees API returned more than 100k entries; shouldn't occur on the test repo).
- Spinner stuck forever — indicates unhandled async error path (violates FR-008).

---

## Scenario US-2: Read a note's full content

**Goal**: Prove FR-004 + FR-005. Click any note and see its rendered markdown with headings, lists, and syntax-highlighted code blocks.

### Steps

1. From the tree populated in Scenario US-1, click `notes/design.md`.
2. Observe the right pane render the note.

### Expected outcomes

- The heading(s) from the markdown appear as semantic `<h1>`/`<h2>` elements, not plain text.
- Any code fences in the file are highlighted with Prism's theme colors.
- Links inside the note are clickable and open in a new tab.
- Front matter is invisible — no YAML dump appears in the reading pane.
- The note title shown in the reading header matches the `title` key from front matter (or falls back to the filename stem if absent).
- Loading time from click to visible content is under 1 second for a note under ~10 KB.

### Edge case to try

Open a note whose file is empty (0 bytes). Per the spec edge cases, the reading pane shows an empty state rather than an error.

---

## Scenario US-3: Browse the tree view interactively

**Goal**: Prove FR-006 + FR-007 + FR-007b + User Story 3 acceptance scenarios.

### Steps

1. Starting from a collapsed tree, click the folder icon next to `projects/`. It expands to reveal `alpha.md` and `beta.md`. Collapse it again.
2. Navigate using keyboard only:
   - Tab focus moves into the tree widget (`role="tree"`).
   - Arrow Down/Up moves between siblings.
   - Arrow Right expands a folder; Arrow Left collapses it.
   - Enter or Space selects a note and loads it in the right pane.
3. With a note selected, observe the browser address bar. It should read something like:
   ```
   https://notes-viewer.pages.dev/your-username/test-repo/notes/projects/alpha.md
   ```
4. Copy that URL into a new tab (bookmark test). The app should load directly to that note without requiring tree interaction.
5. Open a repo that contains subdirectories with no `.md` files (e.g., `assets/images/`). Verify those folders do not appear in the tree — only folders containing at least one `.md` are shown.

### Expected outcomes

- Tree expansion/collapse feels snappy (< 50 ms re-render per the performance goal).
- Keyboard navigation reaches every node; screen reader announces `role="tree"`, `aria-expanded`, and `aria-selected` correctly.
- URL updates reflect the full `/{owner}/{repo}/{path}` structure and persists across reloads.
- An empty folder that genuinely has no notes is hidden rather than shown empty (per the construction rules in data-model.md and github-tree.json).

### Fail signals

- Arrow keys do not move focus inside the tree (accessibility violation of Principle VIII).
- Reloading the bookmarked URL shows the repo input pre-filled but does not select the note — URL parsing bug (violates FR-007b).

---

## Scenario: Error handling — repo not found / no access

**Goal**: Prove FR-008 + Edge Case guidance.

### Steps

1. Enter a deliberately nonexistent repo name such as `nonexistent-user-this-will-not-exist/fake-repo-xyz`.
2. Submit.

### Expected outcomes

- The left pane does not show an empty tree; instead a friendly error message appears describing the problem.
- The message is human-readable and actionable (e.g. "Repository not found or you don't have access.").
- No raw API JSON or stack trace is exposed to the user.

Try the same with a valid public repo name but no authentication cookie (log out first). The app should prompt for sign-in per FR-010 rather than showing a generic error.

---

## Scenario: Rate-limit resilience via ETag

**Goal**: Confirm D-002 conditional-request behavior under the authenticated bucket.

### Steps

1. Select a note in the tree.
2. Wait for it to render.
3. Open DevTools → Network tab and filter by the Contents API request URL.
4. Click the same note again (or navigate away and back via URL).
5. Observe the second request include an `If-None-Match` header matching the `etag` value from step 3.

### Expected outcomes

- Second request returns HTTP 304 instead of 200 (no body transferred).
- Console shows no rate-limit warnings.
- The note content in the reading pane remains identical.

---

## Scenario: Empty repo / no notes

**Goal**: Prove the empty-state branch from the Edge Cases section.

### Steps

1. Point the app at a valid repo that contains no `.md` files (a README-only repo is sufficient).

### Expected outcomes

- Left pane displays a clear empty-state message such as "No notes found in this repo." rather than a blank panel or error.
- Right pane remains idle.

---

## Sign-off template

After running all scenarios, fill this in to close validation:

```
Run date: ______________
Browser: ______________
Deploy URL: ______________
Passed: □ US-1  □ US-2  □ US-3  □ Error  □ ETag  □ Empty repo
Notes: ___________________________________________________________
```

Any failed scenario should be filed as an issue on the feature branch before marking the spec ready for `/speckit-tasks`.
