<!-- SYNC IMPACT REPORT — temporary, remove before commit
  Version change: 1.2.0 → 1.3.0
  Modified principles: III. Text-File Based Storage — expanded rationale (prior amendment, retained)
  Added principles: VI. Leverage Common Formats (no re-inventing the wheel; prefer JSON/YAML/Markdown)
  Added sections: none
  Removed sections: none
  Deferred TODOs: none
-->

# Org-SK Constitution

## Core Principles

### I. Simplicity
Org-SK is simple by design. Every feature, file, and interaction must be explainable in one sentence. Complexity is permitted only when simplicity is genuinely impossible, and then it must be documented alongside the change. YAGNI: do not build what is not needed today.

### II. Markdown-Native Format
All content is written in Markdown. Notes, tasks, and specifications are plain text files structured with headings, lists, and code blocks. No proprietary formats, no binary databases. Files must be human-readable without any tool.

### III. Text-File Based Storage
Org-SK stores everything as text files on disk. The file system is the database; the directory layout is the schema. Files are the unit of ownership, versioning, and portability. Tools read and write text; they never serialize state into opaque blobs. Notes are text-based because they must stand the test of time: plain text survives format churn, tool abandonment, and platform lock-in that proprietary formats do not. A note written today should be readable by any future tool, or by a human with nothing but a text editor.

### IV. User Ownership of Data
The user owns their notes, tasks, and information — completely and exclusively. No data leaves the user's machine without explicit consent. Org-SK is a tool, not a platform: it does not claim, sync, or monetize user content. Privacy is a non-negotiable default, not an opt-in feature.

### V. Deliberate Growth
Small changes to the spec can have huge consequences, so think small — like Basecamp and 37signals. Every change to the spec must be deliberate and thought out before it is made. Org-SK grows at turtle status: features are added slowly, one at a time, and only when there is a real need. A feature that is not clearly justified does not get built. Speed is never an argument for scope.

### VI. Leverage Common Formats
Org-SK follows common best practices and does not re-invent the wheel. Where a format is already standard — JSON, YAML, Markdown, and the like — Org-SK uses it instead of inventing its own. If users are already doing something, or something is already popular, Org-SK tries to leverage it rather than design around it. A custom format is justified only when no common one fits; then the reason is written down.

## Data & File Conventions
- Files use UTF-8 encoding with LF line endings.
- Every note and task lives in a plain `.md` file.
- Metadata, when needed, lives inline as YAML front matter — never in a separate database.
- Directory structure is flat or lightly nested; deep hierarchies are a code smell.
- Backups are the user's responsibility; Org-SK never deletes or overwrites without a visible diff.
- Use standard formats wherever one exists; invent a custom format only when no common one fits, and record why.

## Development Workflow
- Changes are proposed as Markdown specs first, then implemented.
- Every change to behavior is accompanied by a note describing why.
- Tests, when they exist, are text files too — readable, diffable, and versioned with the code.
- Review is a human act: a second person reads the diff and the note before anything is merged.
- No automated deployment; the user decides when and how to ship.

## Governance
This constitution is the highest authority for Org-SK. It supersedes all other practices, docs, and conventions. Any conflict between documents is resolved in favor of this constitution.

Amendments: A proposal is written as a Markdown spec describing the change and its rationale. It is reviewed by a second person, and ratified by the user. Amendments take effect on ratification and are recorded with a version bump.

Versioning: Semantic versioning (MAJOR.MINOR.PATCH). MAJOR for removals or reversals of principles; MINOR for new principles or materially expanded guidance; PATCH for clarifications and wording fixes.

Compliance: Every PR and review must verify alignment with this constitution. Complexity that deviates from a principle must be justified in writing alongside the change.

**Version**: 1.3.0 | **Ratified**: 2026-11-15 | **Last Amended**: 2026-11-15