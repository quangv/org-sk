# Research: GitHub Notes Viewer

**Spec**: [spec.md](./spec.md)
**Date**: 2026-11-18
**Status**: Complete — all NEEDS CLARIFICATION items resolved

---

## D-001 Tree enumeration: Git Trees API over GraphQL / Contents per-directory

**Decision**: Use the REST `GET /repos/{owner}/{repo}/git/trees/{sha}?recursive=1` endpoint to fetch the entire repo tree in one request, as the primary mechanism for building the left pane.

**Rationale**: The recursive trees endpoint returns the full directory hierarchy (up to 100 000 entries, ~7 MB payload) in a single call with each node carrying its path, type (`blob` or `tree`), size, and SHA. A personal notes repo has at most a few hundred files, so the overhead is negligible and pagination is not needed. GraphQL's `repository.tree.entries` connection tops out at 100 items per page and requires manual recursion through sub-trees using cursor pagination — dozens of round trips for the same result. The Contents API per-directory approach would need N+1 calls proportional to folder depth. Only the Git Trees API gives the complete structure atomically.

**Alternatives considered**:
- **GraphQL tree query**: More flexible and typed, but paginate-heavy. Rate limit is a separate bucket (~5 000 points/hour billed by complexity); a deep walk burns cost fast. No HTTP caching support either — no ETag/304s.
- **Contents API per-directory**: Simpler shape but explodes with folder count; each folder requires a new request. Worse on both request count and end-to-end latency.
- **`raw.githubusercontent.com`**: Cheap CDN read for public repos, but never accepts auth headers — useless for private repos, which this app must support via OAuth.

**Sources**: [Git Trees REST API](https://docs.github.com/rest/git/trees); [GraphQL rate limits](https://docs.github.com/en/graphql/overview/rate-limits-and-query-limits-for-the-graphql-api); community report on raw domain refusing auth (Bazarr #3057).

---

## D-002 File content fetching: Contents API with conditional requests

**Decision**: Fetch note body via `GET /repos/{owner}/{repo}/contents/{path}`; prefer the default JSON media type (base64-encoded `content` field) because the client-side JS code does not already depend on a base64 decoder. For large notes (>10 KB), consider streaming the raw media type instead. Store the returned `etag` response header keyed by repo + path and send it back as `If-None-Match` on subsequent reads within the session; an authenticated 304 is free. Also check `X-RateLimit-Remaining` every request.

**Rationale**: The Contents API is purpose-built for file-by-file fetch and supports conditional requests natively. GitHub documented that authenticated conditional responses (304) do not consume the primary rate limit bucket — unauthenticated ones do (observed in live testing), so this optimization only matters after login. Both endpoints (`/git/trees` and `/contents`) expose `cache-control: public, max-age=60, s-maxage=60`, so even browser/CDN-level caching works without custom logic.

Rate limit posture after authentication: 5 000 req/hr primary; ~900 pts/min burst; ~80 content-generating requests/min secondary ceiling. A typical user browsing 50 files per session consumes well under the hourly budget.

**Alternatives considered**:
- **Raw media type (`Accept: application/vnd.github.v3.raw`)**: Avoids client-side base64 decode but still lands in the authenticated bucket and has identical ETag semantics. The default JSON is simpler since the browser decodes from scratch anyway.
- **No conditional requests**: Would work but burns the authenticated bucket twice as fast on revisit — acceptable at this scale, but the ETag path costs nothing extra and buys resilience if the feature scales later.

**Sources**: [Repository Contents REST API](https://docs.github.com/rest/repos/contents); GitHub blog changelog on May 2025 unauthenticated limit tightening; Casey West analysis on rate limits.

---

## D-003 Default branch resolution

**Decision**: One initial `GET /repos/{owner}/{repo}` request during sign-in or first load; read `default_branch` from the response object (a plain string like `"main"`). Cache it per session. Re-read on refresh or if the repo changes.

**Rationale**: Single cheap request returning a single-field answer. The alternative — a bare HEAD against `/refs/heads/main` that follows a 301 — is technically sound but introduces a redirect hop and an extra parse just to discover a value you'll need anyway for other metadata (description, visibility).

**Sources**: [Get a repository](https://docs.github.com/rest/repos/repos#get-a-repository).

---

## D-004 Hosting: Cloudflare Pages + Workers on the free tier

**Decision**: Deploy static frontend assets (HTML/CSS/JS) to **Cloudflare Pages** and route the tiny serverless function(s) — the OAuth token-exchange proxy and the GitHub API proxy (so the secret never reaches the browser) — through **Cloudflare Workers**. One account, no vendor lock-in beyond the platform, zero cost until ~100 k total requests/day across Pages + Workers combined (hard break at 1027, not a soft throttle).

**Rationale**: Vercel Hobby is generous (1 M edge requests, 100 GB transfer/mo) but strictly non-commercial — a solo founder monetizing the app later risks suspension. GitHub Pages is static-only and cannot hold the OAuth secret exchange. Cloudflare Pages Functions count against the same daily quota as Workers, keeping everything simple. PKCE support was added by GitHub in July 2025, but a `client_secret` is still required for the standard web-app token exchange (`POST /login/oauth/access_token`), so a server component is unavoidable regardless of host.

Cost fit: under normal usage this stays forever on the free tier. That is exactly what Principle VII (Cost-Consciousness) demands — pay only when there is real demand.

**Alternative rejected**:
- **Vercel Hobby**: Strong candidate otherwise; the non-commercial restriction is disqualifying for a product the founder intends to ship publicly, even if free today.
- **Self-hosting on a $5 VPS**: Meets Principle VII's "free software" ideal, but adds ops overhead (uptime monitoring, TLS renewal, backup strategy) inconsistent with Principle I (Simplicity: explainable in one sentence) for a solo project with no users yet.

**Sources**: Cloudflare Workers & Pages limits docs; Vercel Hobby terms; GitHub blog on PKCE (July 2025).

---

## D-005 OAuth flow: GitHub OAuth App + HttpOnly cookie for the token

**Decision**: Register a **GitHub OAuth App** (not a GitHub App) with authorization-code + PKCE flow. The OAuth App is the right primitive for a solo user authenticating their own identity against their own repos — fine-grained permissions are unnecessary, short-lived tokens would complicate the reading experience, and GitHub Apps require private-key management that over-engineers a personal tool.

The token exchange happens server-side in a Cloudflare Worker that stores `GITHUB_CLIENT_ID` / `GITHUB_CLIENT_SECRET` as environment variables. The worker issues an `HttpOnly, Secure, SameSite=Lax` cookie containing the access token for subsequent same-origin API calls. Never store the token in `localStorage` — OWASP guidance explicitly flags it as XSS-accessible.

Scopes: `repo` for private repo access (the only scope covering private repo contents on an OAuth App; no narrower read-only variant exists there). Public-only mode would use `public_repo`; the spec requires private repos, so `repo` is the minimal correct scope.

**Rationale**: PKCE defends against authorization-code interception; the server-side secret exchange defends against client-side disclosure. The cookie model keeps the token opaque to page JavaScript, satisfying both security and simplicity.

**Alternatives considered**:
- **GitHub App installation**: Provides fine-grained `Contents: Read` permission (read-only vs the `repo` scope's read+write). Requires storing a PEM private key and generating installation JWTs. Overhead disproportionate for a single-user feature. Worth revisiting only if the app grows to serve multiple tenants.
- **Device Flow**: Built for input-constrained clients (CLI, TV) — needs a code-paste step in the browser. Not suitable for this reading-view UX.
- **Token in localStorage**: The simplest implementation path but violates the "privacy is a default, not an opt-in feature" posture of the constitution and OWASP guidance on XSS.

**Sources**: [OAuth App scopes](https://docs.github.com/en/apps/oauth-apps/building-oauth-apps/scopes-for-oauth-apps); [PKCE changelog](https://github.blog/changelog/2025-07-14-pkce-support-for-oauth-and-github-app-authentication); OWASP Storage guidelines.

---

## D-006 Frontend stack: vanilla JS + CDN deps, no framework

**Decision**: Ship plain HTML/CSS/JS loaded from CDN bundles — no build step, no JSX, no bundler. Client-side routing via `pushState`/`onhashchange` (≈ 30 lines) for `/{owner}/{repo}/{path}` URLs. Load dependencies from CDN: `marked` (markdown parser), `DOMPurify` (sanitize output), `Prism` (syntax highlighting), `vanillatree` (accessible tree widget), `gray-matter` (front matter splitter).

**Rationale**: The app's state model is flat (current repo + selected path → tree + rendered note). There is no shared mutable state between components, no virtual DOM overhead to gain, and no complex component hierarchy. Adding React/Preact/Svelte introduces a build toolchain, runtime dependency, and mental model overhead that outweighs the marginal DX benefit for this scope. Simplicity principle holds: one sentence.

Bundle size budget: marked + DOMPurify + Prism core ≈ 40 KB gzipped; vanillatree adds 3.4 KB. Under 50 KB total — fast load on any metered connection.

Accessibility: WAI-ARIA treeview pattern (role="tree", roving tabindex, arrow-key navigation, aria-expanded, aria-selected) is current as of 2026. `vanillatree` implements it headlessly, so keyboard and screen-reader behavior is covered without framework baggage. Semantic headings in the rendered markdown are preserved by the renderer, making the outline navigable.

**Alternatives considered**:
- **Preact (~3 KB gzipped)**: Only if a React-style component library becomes necessary later. No signal now.
- **Svelte/Lit**: Require build steps — inconsistent with the no-build philosophy.
- **shadcn-tree or Radix Tree**: Excellent accessibility guarantees but React-bounded. Overkill.

**Sources**: [marked docs](https://marked.js.org); [safe-marked](https://github.com/azu/safe-marked); WAI-ARIA 1.2 treeview pattern; [vanillatree](https://github.com/finom/vanillatree); [gray-matter](https://github.com/jonschlinkert/gray-matter).

---

## D-007 Front matter handling

**Decision**: Use `gray-matter` (browser bundle) to split front matter from markdown body at render time. Treat YAML front matter as metadata only — never rendered as note content per the Data & File Conventions.

**Rationale**: `gray-matter` is the de facto standard (used by Astro, VitePress, Gatsby, Netlify) and works in the browser. Caveat: its `js` engine uses `eval()` — avoid that engine; only YAML is supported in the browser bundle, which is sufficient here. The regex-splitter alternative (`/^---\s*[\s\S]*?---\s*/`) plus `js-yaml` is heavier and reinvents a solved problem.

**Alternatives considered**:
- **Custom regex + js-yaml**: Heavier bundle, more maintenance burden, same outcome.
- **Skip front matter entirely**: Violates the convention stated in the spec's Assumptions section.

---

## Open Items

None — all previously open technical questions are resolved above.
