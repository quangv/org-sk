# Feature Specification: GitHub Notes Viewer

**Feature Branch**: `001-github-notes-viewer`

**Created**: 2026-11-15

**Status**: Draft

**Input**: User description: "there should be a website/webapp that lets me see/read all my notes I have in a github repo."

## Clarifications

### Session 2026-11-16

- Q: What is the scope of search and filter, and is there any additional UI element you want added? → A: Search and filter (User Story 3) is out of scope. Add a tree view on the left side of the screen to show notes and folders in the repo.
- Q: With the tree view added on the left, what happens to the note list from User Story 1? → A: Two-pane layout: tree view on the left, reading view on the right. The tree replaces the flat list as the primary navigation; clicking a note in the tree opens its content in the right pane, and the URL updates so the user can bookmark the note.

## User Scenarios & Testing *(mandatory)*

<!--
  User stories are prioritized as journeys ordered by importance. Each is independently testable.
  P1 = core value, P2 = needed for a complete reading experience, P3 = polish for scale.
-->

### User Story 1 - List all notes from a GitHub repo (Priority: P1)

As a user, I want to enter the name of a GitHub repository that holds my notes and see every note in the tree view on the left, so that I can review everything I have written without cloning the repo or opening raw files.

**Why this priority**: Listing is the core purpose of the feature. Without it the webapp has no value, and it is independently usable on its own.

**Independent Test**: Point the app at a GitHub repo containing markdown notes; verify every note appears in the tree view.

**Acceptance Scenarios**:

1. Given a GitHub repo with 3 note files, When I enter the repo name, Then all 3 notes appear in the tree view.
2. Given a repo with note files in subdirectories, When I enter the repo name, Then notes in subdirectories appear under their folders in the tree view.

---

### User Story 2 - Read a note's full content (Priority: P2)

As a user, I want to click a note in the tree view and read its full contents in a clean reading view on the right, so that I can actually read what I wrote without dealing with raw markdown source.

**Why this priority**: Reading is the point of "see/read" — listing alone delivers only half the value. It is independently testable and turns the tree into a usable product.

**Independent Test**: Click a note in the tree view; verify its full content renders in the reading view.

**Acceptance Scenarios**:

1. Given a note in the tree view, When I click it, Then the full note content opens in the reading view on the right.
2. Given a note with markdown formatting, When I read it, Then headings, lists, and code blocks render readably.

---

### User Story 3 - Browse notes and folders in a tree view (Priority: P2)

As a user, I want to see a tree view on the left side of the screen that shows the notes and folders in the repo, so that I can navigate by folder structure instead of only by a flat list.

**Why this priority**: Navigation is part of a complete reading experience. The tree makes the repo's structure visible and lets the user pick a note by where it lives, which matters once notes spread across folders. It is independently testable and pairs with Stories 1 and 2.

**Independent Test**: Point the app at a repo with notes in subdirectories; verify the left panel shows folders and the notes inside them, and clicking a note in the tree opens its content.

**Acceptance Scenarios**:

1. Given a repo with notes in subdirectories, When I view the app, Then the left panel shows each folder and the notes inside it.
2. Given a note in the tree, When I click it, Then its full content opens in the reading view on the right.
3. Given an empty folder, When I view the tree, Then the folder is shown with no notes under it, rather than hidden or erroring.

---

### User Story 4 - Search and filter notes (Out of Scope)

> **Out of scope** — confirmed 2026-11-16. Search and filter is not part of this feature. It is deferred under Principle V (Deliberate Growth): features are added one at a time, only when there is a real need, and a feature that is not clearly justified does not get built. It will be reconsidered as a separate, later feature.

As a user, I want to search my notes by title or content keyword, so that I can find a specific note quickly once I have many of them.

**Why this priority**: Valuable at scale, but not required for the first usable version. It builds on Stories 1 and 2 and can ship independently afterward. It is deferred under Principle V (Deliberate Growth): features are added one at a time, only when there is a real need, and a feature that is not clearly justified does not get built. Search is justified but is not needed for the first usable version, so it is scoped out of v1 rather than rushed in.

**Independent Test**: Type a keyword that appears in one note's title; verify only matching notes appear in the list.

**Acceptance Scenarios**:

1. Given 10 notes, When I type a keyword, Then only notes matching that keyword appear in the list.
2. Given no matches, When I type a keyword, Then the list shows a clear "no notes found" message instead of an empty screen.

---

### Edge Cases

- What happens when the repo is empty or has no note files? → Show an empty state with a clear message, not an error.
- What happens when the repo does not exist or the user has no access? → Show a friendly error message explaining the problem.
- What happens when a note file is very large? → Render it; do not crash or silently truncate.
- What happens when the network request to GitHub fails or times out? → Show a retryable error message.

## Requirements *(mandatory)*

### Functional Requirements

- **FR-001**: System MUST let the user specify a GitHub repository by owner and name (or a full repository URL).
- **FR-002**: System MUST retrieve the list of note files from the specified repository.
- **FR-003**: System MUST display the retrieved notes as a list showing each note's title.
- **FR-004**: System MUST let the user select a note from the list and display its full content in a reading view.
- **FR-005**: System MUST render markdown content so headings, lists, code blocks, and links are readable.
- **FR-006**: System MUST provide a tree view on the left side of the screen that shows the folders and note files in the repo, mirroring the repository's directory structure.
- **FR-007**: System MUST let the user select a note from the tree view and display its full content in the reading view.
- **FR-007b**: System MUST update the browser URL when a note is opened so the user can bookmark it and return to that note directly. The URL must encode the repository and the note's path.
- **FR-008**: System MUST show a clear error message when the repository cannot be accessed or notes cannot be loaded.
- **FR-009**: System MUST NOT require the user to clone the repository locally to use the webapp.
- **FR-010**: System MUST let the user authenticate with GitHub so their private repositories are readable. The user grants the app access to their repos through an OAuth flow; the app then reads note files from the repos the user selects.

### Key Entities *(include if feature involves data)*

- **Note**: A markdown file in the user's GitHub repo. Key attributes: title (derived from file name), path within the repo, content, last modified date.
- **GitHub Repository**: The source of the notes. Key attributes: owner, name, default branch.
- **Reading View**: The view that displays a single note's rendered content.
- **Tree View**: The left-side panel that displays the repository's folders and note files in a hierarchical structure. Key attributes: folder nodes, note nodes under each folder, selected state.

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: A user can enter a repo name and see all of its notes listed within a few seconds of a single interaction.
- **SC-002**: A user can open any note and read its full content with one click from the tree view, and the opened note is bookmarkable via its URL.
- **SC-003**: The tree view on the left shows every folder and note in the repo, and clicking a note in the tree opens its content in the reading view.
- **SC-004**: When a repo cannot be loaded, the user sees a clear error message instead of a broken or blank screen.
- **SC-005**: The webapp runs without requiring the user to install or clone anything beyond opening it in a browser.

## Compliance Note

This feature uses a front-facing hosted website, so note content — including from private repositories — transits through the webapp's servers rather than staying on the user's machine. This is **not a deviation** from Principle IV (User Ownership of Data): the principle permits data to leave the machine when the user gives **explicit consent**, and the user explicitly chose the hosted model.

It is a higher-privilege data path than a local-only tool, so it is documented here and must be re-stated in any future PR or review for this feature, per the constitution's compliance requirement. The app must also respect the rest of Principle IV: it must not claim, sync, or monetize user content, and privacy remains a default rather than an opt-in feature.

## Assumptions

- Notes are stored as Markdown (.md) files in the GitHub repo; other file types are out of scope for v1.
- The webapp is a front-facing website served over the internet. Note content (including from private repos) will transit through the webapp's servers. This is a deliberate deviation from the constitution's User Ownership of Data principle (IV), which normally treats data leaving the user's machine as requiring explicit consent. The user has explicitly chosen this hosted model.
- The user has a modern browser with internet access to reach GitHub and the webapp.
- Note titles are derived from file names; YAML front matter, if present, is treated as metadata per the Data & File Conventions and is not rendered as note content.
- The repo's default branch is the source of notes; the app does not browse other branches.