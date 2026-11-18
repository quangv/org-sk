# Data Model: GitHub Notes Viewer

**Spec**: [spec.md](./spec.md) | **Research**: [research.md](./research.md) | **Plan**: [plan.md](./plan.md)
**Date**: 2026-11-18 | **Phase**: 1 (design)

All entities are transient — held in client-side memory for a single browser session only. Nothing is persisted to `localStorage`, IndexedDB, or a server database. This matches the storage posture documented in D-004 and D-005.

---

## Entity: GitHubRepository

Source of truth comes from `GET /repos/{owner}/{repo}` (D-003). The app reads one record and caches it per session so it does not re-request when the user navigates between notes in the same repo.

| Field | Type | Required | Source | Notes |
|---|---|---|---|---|
| `owner` | string | yes | FR-001 input; API response | Canonical lowercase login/name |
| `name` | string | yes | FR-001 input; API response | Repository slug |
| `defaultBranch` | string | yes | `GET /repos/{owner}/{repo}` → `default_branch` | Cached after first fetch; not re-fetched on navigation |
| `visibility` | `"public" \| "private"` | yes | Same endpoint | Drives auth requirement: private repos require OAuth cookie |
| `description` | string? | no | Same endpoint | Displayed in page header if present |
| `lastFetchedAt` | ISO-8601 timestamp | yes | Local clock at time of fetch | Used for staleness heuristics (optional polish) |

**Validation rules** (from FR-001, FR-008):
- `owner` and `name` must each be non-empty after trimming.
- If the API returns a 404 or the user lacks permission, the repository entity is NOT created; instead an error message is shown per Edge Cases.
- The app accepts either a bare `owner/repo` string or a full `https://github.com/owner/repo` URL and normalizes both to `{owner, name}` before calling any API.

**State transitions**: None — read-only. One lifecycle: *uninitialized → loaded → (stale)*. Staleness is a UI signal, not a distinct state.

---

## Entity: Note

Represents a single `.md` file fetched via `GET /repos/{owner}/{repo}/contents/{path}` (D-002). Stored as an in-memory map keyed by `(owner, repo, path)` so that revisiting a bookmarked note within the same session avoids a second network request if the ETag was cached.

| Field | Type | Required | Source | Notes |
|---|---|---|---|---|
| `path` | string | yes | Content of file path returned by tree or manual entry | Forward-slash relative to repo root; e.g. `docs/design.md` |
| `filename` | string | yes | Basename of `path` | Used as title fallback when front matter omits `title` |
| `title` | string | yes | YAML front matter `title` key, else `filename` stem without extension | Per spec Assumptions + Data & File Conventions |
| `content` | string | yes | Base64-decoded `content` field from Contents API response | Raw markdown text; never stored long-term |
| `frontMatter` | object? | no | Parsed YAML block from `gray-matter` (D-007) | Treated as metadata only — never rendered as note content |
| `sha` | string | yes | SHA-256 hash of the content after decode | Matches the blob SHA from the tree |
| `size` | number | yes | `size` field from API response in bytes | Used for the empty-state heuristic |
| `lastModified` | ISO-8601 timestamp | yes | Git commit date at HEAD of this path | Displayed in reading view |
| `etag` | string | yes | `ETag` response header from Contents API | Sent back verbatim as `If-None-Match` on subsequent fetches |
| `fetchedAt` | ISO-8601 timestamp | yes | Local clock at fetch time | Enables stale-while-revalidate behavior |

**Validation rules** (FR-002, FR-004, FR-008, Edge Cases):
- `path` must end with `.md`; non-markdown files are skipped silently during tree population (not surfaced as errors — they are simply not nodes in the tree).
- `size > 0` must hold; a zero-byte file renders as an empty reading pane rather than an error, matching the empty-state guidance.
- On 304 (Not Modified), the cached `Note` object is returned without mutation except updating `fetchedAt`.
- If the Contents API returns a non-2xx code other than 304, the note is considered unloaded; the reading view shows the retryable error per FR-008.

**Relationships**:
- Belongs to exactly one `GitHubRepository` (the `owner`/`name` pair).
- Appears as a leaf node in exactly one `TreeNode` (of type `"blob"`).
- Has zero or more related notes via `frontMatter.links` (a common convention in personal knowledge repos); no model enforcement required yet — handled as rendering convention later.

---

## Entity: TreeNode

Represents a single node in the left-pane tree. Constructed once from the recursive output of `GET /repos/{owner}/{repo}/git/trees/{sha}?recursive=1` (D-001) and rebuilt only when the repo changes. Each node carries a flat list shape so the widget can render lazily; parent pointers are used internally for tree construction, not stored as fields.

| Field | Type | Required | Source | Notes |
|---|---|---|---|---|
| `id` | string | yes | Derived: normalized path with `/` replaced by `-` (root = `root`) | Stable identifier across rebuilds for the same subtree; used as ARIA node ID |
| `label` | string | yes | Basename of path; directory names are used as-is, including dotfiles | Shown to the user in the tree widget |
| `path` | string | yes | Full forward-slash path from repo root | Identity key into the `Note` map when type is `blob` |
| `type` | `"tree" \| "blob"` | yes | `type` field from GitHub Trees API | Determines whether the node is expandable |
| `children` | `TreeNode[]`? | no (absent on leaves) | Children of `tree` nodes only | Populated recursively; `undefined` on leaf blobs |
| `sha` | string | yes | SHA from GitHub Trees API | Used to detect staleness; not displayed |
| `size` | number | yes | Size in bytes (only meaningful for blobs) | Skipped for directories |
| `isFolder` | boolean | derived | `type === "tree"` | Convenience accessor for rendering; not duplicated in storage |
| `isSelected` | boolean | no | Set by user click; managed in component state | Transient UI state, not part of the persistent model |
| `isExpanded` | boolean | no | Defaults to false for all but the root segment of the active path | Managed by the tree widget; reset on repo change |

**Tree construction algorithm** (applied once after successful trees API call):
1. Flatten the recursive response into a list of `{path, type, sha, size}` entries.
2. Sort by `path` lexicographically so sibling order mirrors GitHub's native ordering.
3. Group entries by directory prefix to build parent→children relationships.
4. Omit any entry whose path does not end with `.md` when constructing leaf nodes — these are not represented in the tree (non-note blobs do not become tree items). However, intermediate directories that contain only non-note blobs still appear as folders so the structure is visible.

> **Design note**: Directories that contain *no* `.md` children are intentionally omitted from the tree. This keeps the left pane focused on what the user came for (notes) while still exposing the folder structure for notes that actually exist. See D-001 rationale for why we don't enumerate every directory.

**Validation rules** (FR-003, FR-006, FR-007, US-3 edge case):
- An empty folder that contains no `.md` files is hidden (per the algorithm above). If the user has configured their repo so some non-note files appear next to notes (e.g. images referenced in markdown), those are irrelevant to tree construction since the images live inside the rendered note content, not as tree siblings.
- If the trees API returns an empty tree (`truncated: true` or zero entries), the app treats it as "repo has no notes" and shows the empty state per Edge Cases — not an error.

---

## Entity: ReadingViewState

Holds the transient UI state around the right-pane reading view. Distinct from the `Note` entity because it tracks user interaction state rather than content.

| Field | Type | Required | Notes |
|---|---|---|---|
| `activeNotePath` | string? | yes | `null` when no note is selected; set when user clicks a tree node or the URL is parsed |
| `note` | Note? | yes | The fully populated note object; `null` until the first fetch completes |
| `renderedHtml` | string? | yes | Output of `DOMPurify.sanitize(marked(note.content))` — cached per fetch so re-render on expand/collapse doesn't re-parse |
| `isLoading` | boolean | yes | True while the fetch is in flight; drives the spinner |
| `error` | string? | yes | Human-readable error message set on non-304 failure; cleared on success |
| `retryCount` | number | yes | Incremented on each retry button click; resets when a different note is selected |

**State machine** (simplified):

```
initial ──► loading (fetch in flight)
   │
   ├──► loaded      (200 OK or 304)
   │     │
   │     └──► switching (user clicked another note)
   │           └──► loading …
   │
   └──► error       (4xx/5xx other than 304)
         │
         └──► retrying → loading …
```

No persistence across tab close. The entire `ReadingViewState` is discarded when `app.js` initializes on page load.

---

## Relationship Diagram

```
GitHubRepository ────────┐
                         ├──► TreeNode[]  (constructed from tree API response)
                         │        │
                         │        └──[path]──► Note (one-to-one when type=blob & .md suffix)
                         │
                         └──[defaultBranch.sha]──► Git tree (transient, not a separate entity)

ReadingViewState ◄── activeNotePath ──► Note
ReadingViewState.renderedHtml   ←─ marked(DOMPurify(Note.content))
```

Only three first-class entities exist in memory: `GitHubRepository`, `Note`, and `TreeNode`. `ReadingViewState` is the fourth, but it is purely UI state and has no relation outside the current tab.

---

## Validation Rules Summary (traceable to requirements)

| Rule | Requirement | How enforced |
|---|---|---|
| Repo owner/name non-empty | FR-001 | Input validation before normalization |
| Non-.md files excluded from tree | Spec Assumptions | Path suffix filter during tree construction |
| Empty folder omission | US-3 edge case + Plan note | Directory pruning in construction algorithm |
| Token never touches localStorage | FR-010 + Principle IV exception | HttpOnly cookie set server-side (see research D-005) |
| Front matter treated as metadata only | Spec Assumptions + D-007 | `gray-matter` return value used solely for `title` extraction |
| ETag sent back on repeat fetch | D-002 | Stored on `Note.etag`, reused on next fetch |
| 304 does not mutate note body | D-002 | Branch in fetch logic preserves existing `Note` except timestamp |
