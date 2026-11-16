# Org-SK

Org-SK is a simple, Markdown-native, text-file-based tool for notes and tasks.

## Constitution

The project constitution lives at `.specify/memory/constitution.md` and is the
highest authority for this project. It supersedes all other docs and
conventions. Read it before making decisions that touch scope, format, storage,
privacy, or cost.

Key posture: solo-founder, free/cheap by default — see Principle VII
(Cost-Consciousness). Any hosting, service provider, or license choice must
consider cost first.

## Development Workflow

- Changes are proposed as Markdown specs first, then implemented.
- Every change to behavior is accompanied by a note describing why.
- Tests, when they exist, are text files too — readable, diffable, versioned.
- Review is a human act: a second person reads the diff and the note before
  anything is merged.
- No automated deployment; the user decides when and how to ship.