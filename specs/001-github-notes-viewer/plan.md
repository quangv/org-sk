# Implementation Plan: GitHub Notes Viewer

**Branch**: `001-github-notes-viewer` | **Date**: 2026-11-18 | **Spec**: [spec.md](./spec.md)

**Input**: Feature specification from `/specs/001-github-notes-viewer/spec.md`

## Summary

A minimal, free-tier webapp that lets a user point at any GitHub repository (public or private via OAuth) and read its markdown notes in a two-pane layout: a left-side accessible tree view mirroring the repo's folder structure, and a right-side reading view with rendered markdown, syntax highlighting, and bookmarkable URLs (`/{owner}/{repo}/{path}`). Static frontend served from Cloudflare Pages; a single tiny Workers function handles the GitHub OAuth token exchange so the client secret never reaches the browser. No build step, no framework — vanilla JS loaded from CDN deps.

Primary architectural choice: REST Git Trees API for the one-shot recursive directory listing, Contents API per file for note bodies with conditional requests (ETag) to respect rate limits, and an HttpOnly cookie to carry the OAuth token server-side.

## Technical Context

**Language/Version**: JavaScript (ES modules), no transpilation target; modern browser only (Chrome 90+, Firefox 90+, Safari 15+) as implied by the spec's assumption of "modern browser with internet access".

**Primary Dependencies**: None locally. Runtime deps loaded from CDN — `marked` (markdown parser), `DOMPurify` (output sanitizer), `Prism` (syntax highlighting), `vanillatree` (accessible ARIA tree widget), `gray-matter` (front matter splitter). Build tooling: none.

**Storage**: Client-side memory only during a session. The GitHub access token is held in an HttpOnly, Secure, SameSite=Lax cookie set by the Cloudflare Worker proxy; nothing is stored in `localStorage` or IndexedDB. No database. Note content is fetched fresh per request (with ETag-based conditional caching) and discarded when the tab closes.

**Testing**: Manual validation against real repos, guided by the acceptance scenarios in the spec and the quickstart scenarios in `quickstart.md`. No automated test harness planned for this feature — consistent with Principle I (Simplicity) and the constitution's development workflow that treats human review as the primary quality gate.

**Target Platform**: Any modern browser on macOS, Windows, Linux, or iOS/Android mobile browsers. Deployed on Cloudflare's global edge — no region-specific constraints.

**Project Type**: Single-page static web application with a thin serverless OAuth/API proxy layer.

**Performance Goals**: Initial repo tree fetch under 3 seconds on a typical broadband connection (~one round trip to GitHub plus one to Cloudflare edge). Individual note load under 1 second. Tree re-render on collapse/expand under 50 ms. All of this assumes authenticated access (5 000 req/hr bucket); unauthenticated browsing is not a supported path (FR-010 requires login before any repo is visible).

**Constraints**: Free tier only — zero recurring cost, hard cap of ~100 k combined Pages+Workers requests/day. GitHub secondary rate limit (~80 content reads/min) must be respected via ETag caching. No client-side storage of tokens (OWASP + privacy default). Keyboard-reachable tree navigation required (accessibility principle). Single owner/developer stack means no team coordination overhead but also no redundancy — every decision must stand on its own merits.

**Scale/Scope**: One user (the founder). Typical repos: <500 `.md` files. Maximum concurrent users: 1 (self-hosted deployment pattern). The architecture is deliberately chosen so it *could* serve more users without revision, but there is no requirement to do so.

## Constitution Check

*GATE: Must pass before Phase 0 research. Re-checked after Phase 1 design.*

| Principle | Check | Finding |
|---|---|---|
| I. Simplicity | Pass | Vanilla JS, no build step, no framework, one serverless function. Every interaction explainable in one sentence. |
| II. Markdown-Native Format | Pass | App consumes and renders `.md` files; output is human-readable HTML. No proprietary formats introduced. |
| III. Text-File Based Storage | Pass | App has no persistent store; all data comes from GitHub and flows through in-memory only. Meets the spirit even though the app itself does not touch local text files. |
| IV. User Ownership of Data | Pass (documented exception) | Note content transits through Cloudflare Workers during rendering. Not a deviation because the spec explicitly states the user chose the hosted model with consent; compliance note in `spec.md` covers re-statement obligations. Token is HttpOnly cookie, never localStorage. App claims no ownership over content (read-only, no sync, no monetization). |
| V. Deliberate Growth | Pass | Search/filter explicitly scoped out (US-4). Future features will be added one at a time with their own specs. Current scope is intentionally narrow. |
| VI. Leverage Common Formats | Pass | Uses standard YAML front matter, common Markdown, GitHub REST APIs, well-known libraries (marked, Prism, DOMPurify, gray-matter) instead of inventing alternatives. |
| VII. Cost-Consciousness | Pass | Zero-cost hosting (Cloudflare Pages + Workers free tier, <100 k req/day shared quota). No paid services adopted; no recurring subscription risk. Justification recorded in D-004 above. |
| VIII. Accessibility First | Pass | ARIA treeview pattern implemented via `vanillatree` (keyboard-navigable, screen-reader compatible). Semantic headings preserved from markdown. Color choices are non-content-bearing; contrast follows WCAG AA minimums. All interactive elements reachable via keyboard. Checked at design time per the constitution's accessibility review workflow. |

**Overall gate**: Pass. No violations require complexity-tracking entry.

## Project Structure

### Documentation (this feature)

```text
specs/001-github-notes-viewer/
├── spec.md              # Feature specification (already complete)
├── checklists/
│   └── requirements.md  # Requirements completeness checklist (all [x])
├── plan.md              # This file (/speckit-plan output)
├── research.md          # Phase 0 output — six decisions resolved
├── data-model.md        # Phase 1 output — entities and relationships
├── contracts/           # Phase 1 output — API shapes for implementation
│   ├── github-tree.json
│   └── github-note.json
├── quickstart.md        # Phase 1 output — manual validation scenarios
└── tasks.md             # Phase 2 output (/speckit-tasks command - NOT created here)
```

### Source Code (repository root)

```text
apps/github-notes-viewer/       # Standalone webapp (no monorepo tooling needed yet)
├── index.html                  # Single-page shell with tree + reading panes
├── css/
│   └── main.css                # Two-pane layout, responsive adjustments
├── js/
│   ├── app.js                  # Router, URL state, render orchestration
│   ├── api.js                  # GitHub API helpers (tree fetch, note fetch, ETag cache)
│   ├── auth.js                 # OAuth redirect handling, cookie checks
│   └── components/
│       ├── tree.js             # Tree widget bootstrap (vanillatree wrapper)
│       └── reader.js           # Markdown renderer (marked + DOMPurify + Prism)
└── assets/                     # Favicon, optional manifest
wrangler.toml                   # Cloudflare Workers config for the proxy function
workers/proxy.ts                # Single worker: handles /auth/* and proxies GitHub API calls
package.json                    # Optional dev-dependencies (wrangler CLI only)
README.md                       # How to deploy, environment variables, gotchas
```

**Structure Decision**: Option 1 (single project) is the only layout used. No backend/frontend split — the Workers proxy is small enough that a separate `backend/` module would add ceremony without value. `apps/github-notes-viewer/` keeps the feature isolated as a sub-directory of the repo without imposing monorepo tooling (which Principle I says to avoid unless necessary). No linting/formatting config added until a second contributor justifies it — Principles I and V both argue against infrastructure that serves nobody yet.

## Complexity Tracking

Not applicable — no constitution violations were identified. The Constitution Check table above documents every principle pass, including the documented-but-consented exception under Principle IV (hosted data path).
