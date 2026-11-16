# Feature Specification: GitHub Notes Viewer

**Feature Branch**: `001-github-notes-viewer`

**Created**: 2026-11-15

**Status**: Draft

**Input**: User description: "there should be a website/webapp that lets me see/read all my notes I have in a github repo."

## User Scenarios & Testing *(mandatory)*

<!--
  User stories are prioritized as journeys ordered by importance. Each is independently testable.
  P1 = core value, P2 = needed for a complete reading experience, P3 = polish for scale.
-->

### User Story 1 - List all notes from a GitHub repo (Priority: P1)

As a user, I want to enter the name of a GitHub repository that holds my notes and see every note displayed in a readable list, so that I can review everything I have written without cloning the repo or opening raw files.

**Why this priority**: Listing is the core purpose of the feature. Without it the webapp has no value, and it is independently usable on its own.

**Independent Test**: Point the app at a GitHub repo containing markdown notes; verify every note appears in the list.

**Acceptance Scenarios**:

1. Given a GitHub repo with 3 note files, When I enter the repo name, Then all 3 notes appear in the list.
2. Given a repo with note files in subdirectories, When I enter the repo name, Then notes in subdirectories also appear in the list.

---

### User Story 2 - Read a note's full content (Priority: P2)

As a user, I want to click a note in the list and read its full contents in a clean reading view, so that I can actually read what I wrote without dealing with raw markdown source.

**Why this priority**: Reading is the point of "see/read" — listing alone delivers only half the value. It is independently testable and turns the list into a usable product.

**Independent Test**: Click a note in the list; verify its full content renders in the reading view.

**Acceptance Scenarios**:

1. Given a note in the list, When I click it, Then the full note content opens in a reading view.
2. Given a note with markdown formatting, When I read it, Then headings, lists, and code blocks render readably.

---

### User Story 3 - Search and filter notes (Priority: P3)

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
- **FR-006**: System MUST provide a search field that filters the note list by title and content keywords.
- **FR-007**: System MUST show a clear error message when the repository cannot be accessed or notes cannot be loaded.
- **FR-008**: System MUST NOT require the user to clone the repository locally to use the webapp.
- **FR-009**: System MUST let the user authenticate with GitHub so their private repositories are readable. The user grants the app access to their repos through an OAuth flow; the app then reads note files from the repos the user selects.

### Key Entities *(include if feature involves data)*

- **Note**: A markdown file in the user's GitHub repo. Key attributes: title (derived from file name), path within the repo, content, last modified date.
- **GitHub Repository**: The source of the notes. Key attributes: owner, name, default branch.
- **Reading View**: The view that displays a single note's rendered content.

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: A user can enter a repo name and see all of its notes listed within a few seconds of a single interaction.
- **SC-002**: A user can open any note and read its full content with one click, without leaving the list view.
- **SC-003**: Searching for a keyword returns only matching notes, and the result updates as the user types.
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