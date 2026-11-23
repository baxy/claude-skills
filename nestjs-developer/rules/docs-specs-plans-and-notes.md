---
title: Keep Specs and Plans in docs/, Notes Out of Git
impact: MEDIUM
impactDescription: Every spec and plan is found in one place, paired, dated and linked to its ticket; private notes never reach the repository
tags: documentation, specs, plans, notes
---

## Keep Specs and Plans in docs/, Notes Out of Git

Specs (what a change must do) and plans (how it will be built) live in **one place**: `docs/specs/` and `docs/plans/` at the repository root. Never at the root as `specs/`, never next to the code they describe (`src/order/docs/`), and never in a tool's own default folder. Readers then look in one folder for every change.

**File names: `{yyyy-mm-dd}-{ticket}-{name}.md`, or `{yyyy-mm-dd}-{name}.md` when the work has no ticket**, lowercase and dash-separated:

- `{yyyy-mm-dd}` is the day the file is created. It never changes when the file is edited, so links to it keep working.
- `{ticket}` is the issue tracker's id, lowercased (`ord-261`), whenever the work has one: the branch carries it (`feat/ord-261-refunds`) or the request names it. A spike or a draft before a ticket exists has none, and the name starts with the date and the name only.
- `{name}` says what the change is: `orders-refunds`, `session-cleanup`.
- **A spec and its plan share the file name** when they are written together: `docs/specs/2026-10-29-ord-270-orders-refunds.md` is planned in `docs/plans/2026-10-29-ord-270-orders-refunds.md`. Every plan links its spec on its first line.
- A spec too large for one file becomes a folder with the same name holding numbered files and a `README.md` index listing them: `docs/specs/2026-10-29-ord-280-billing-rewrite/01-current-behavior.md`.

**A spec written before its ticket exists is renamed when the ticket arrives.** When a ticket is created for it, the first change on the ticket's branch renames it to carry the ticket, keeping its date: `docs/specs/2026-10-30-outbox-spike.md` becomes `docs/specs/2026-10-30-ord-290-outbox-spike.md`. The same change updates every link to it in the repository (`grep -rn 2026-10-30-outbox-spike docs/`), and the ticket links the new path afterwards, so the link never breaks. Only links shared before the rename (chat, pull request comments) point at the old path. This is common for an analysis of a legacy system (`legacy-to-clean-specs`), which produces many specs before anyone has created a ticket; its files are renamed inside their folder (below).

**A legacy analysis is one folder.** One folder per analysis, `docs/specs/{yyyy-mm-dd}-{name}/`, holding a `README.md` index (GitHub shows it when the folder is opened) listing every file, and the analysis as numbered files (`01-executive-summary.md`, `02-current-behavior.md`, ...).

**Plugins and skills that write specs or plans use these folders.** `legacy-to-clean-specs` writes the legacy-analysis folder above. The superpowers brainstorming and writing-plans skills write to `docs/superpowers/specs/` and `docs/superpowers/plans/` with a `-design` suffix by default. A project's location overrides those defaults, and this rule is that location. Drop tool-specific suffixes: the folder already says whether the file is a spec or a plan.

**A business choice belongs to the spec, not the code.** When a task leaves one open (what counts as revenue, a limit, what happens at a boundary), ask before building it, and write the answer into the spec. When no one can answer in time, take the choice that is cheapest to change later, write it into the spec as an assumption, and name it first in the report. A choice made only in code is one nobody reviewed, and the next change may make it the other way.

**Notes are private: `docs/notes/{ticket}/` (or `docs/notes/{topic}/` without a ticket), gitignored.** Drafts (a chat message, a pull request description), scratch analysis and anything written for one developer go there. They are never committed, and a committed file never links to one, because nobody else has it. The folder lives in one checkout only: other clones and git worktrees do not have it, and `git clean -X` deletes it. Search tools skip gitignored files, so open a note by its path.

The `docs/specs/` and `docs/plans/` folders exist from the start, each holding a `.gitkeep`.

**Incorrect (code folders, tool defaults, undated names, notes committed):**

```
src/order/docs/refunds.md                                 // next to the code, no date, no ticket
docs/superpowers/specs/2026-10-29-refunds-design.md       // the tool's default folder and suffix
specs/billing-rewrite.md                                  // root specs/ outside docs/
docs/plans/2026-10-29-ord-270-refund-plan.md              // plan named differently from its spec
docs/specs/2026-10-29-orders-refunds.md                   // written on branch feat/ord-270-orders-refunds: ticket missing
docs/specs/notes-chat-draft.md                            // a private draft committed as a spec
```

**Correct (one folder per kind, paired names, private notes ignored):**

```
docs/
├── specs/
│   ├── 2026-10-29-ord-270-orders-refunds.md
│   ├── 2026-10-29-ord-280-billing-rewrite/               // large spec: folder of numbered files
│   │   ├── README.md                                     // the index
│   │   ├── 01-current-behavior.md
│   │   └── 02-target-architecture.md
│   └── 2026-10-30-outbox-spike.md                        // no ticket yet
├── plans/
│   └── 2026-10-29-ord-270-orders-refunds.md              // same name as its spec
└── notes/                                                // gitignored
    └── ord-270/
        ├── chat-handover.md
        └── pr-description.md
```

```gitignore
# .gitignore
docs/notes/
```

Reference: [superpowers brainstorming skill](https://github.com/obra/superpowers) (default spec location, overridable by the project)
