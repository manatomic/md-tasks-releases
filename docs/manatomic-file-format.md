---
title: manatomic file format
type: spec
status: final
related: ["[[MAN-3 - File format spec + tasks-core model, parser and serializer]]", "[[MAN-15 - Roadmap screen milestone Gantt with epic bars and progress]]"]
created: 2026-08-22
updated: 2026-09-15
---

# manatomic file format

- **Status**: v1 (settled by MAN-3)
- **Scope**: the markdown/YAML files under the data root that are the single source of truth for tasks, docs, decisions and project configuration. Read and written by humans (editor, Obsidian), by agents (`manatomic tasks` CLI) and by the UI (server).
- **Implementation**: `packages/tasks-core` (`parseTask`, `parseDoc`, `parseDecision`, `parseConfig` and the matching `serialize*` functions). The fenced examples below tagged `file=...` are parsed by the tasks-core test-suite, so this document and the parser cannot drift apart.

## 1. Data root and layout

The data root defaults to `manatomic/` at the repository root (configurable; it is the umbrella folder shared by future manatomic tools).

```text
manatomic/
  config.yml                     # project configuration (section 6)
  tasks/<ID> - <title>.md        # tasks, subtasks, bugs, spikes and epics
  docs/**/<name>.md              # free-form docs (brainstorms, research, specs)
  decisions/<ID> - <title>.md    # decision records
```

- **Ids** are `<PREFIX>-<n>` (`MAN-12`). Tasks, epics, subtasks, bugs and spikes share the task prefix from `config.yml`; decisions use `DEC-<n>`.
- **Filenames** are `<ID> - <title>.md`. Only the leading id is significant for resolution, so renaming the title part never breaks links.
- **Wikilinks** (`[[MAN-12 - Login form]]`, `[[MAN-12 - Login form|MAN-12]]`) are Obsidian-compatible. Resolution takes the leading id of the link target; the rest of the target and any alias are display text. In YAML they must be quoted (`"[[...]]"`) because `[` starts a flow sequence.

## 2. General file rules

- UTF-8. Line endings are detected per file (`\n` or `\r\n`) and preserved on write, as is the presence or absence of a final newline.
- Every file starts with a YAML frontmatter block delimited by `---` lines. The frontmatter is YAML 1.2 core schema (dates are plain strings, never converted).
- **Passthrough**: unknown frontmatter keys, YAML comments, blank lines inside the frontmatter and unrecognised `##` sections are preserved byte-for-byte. Tools only re-render the specific key, item or entry they change. `serialize(parse(text)) === text` for any input.
- **Errors never throw.** Parsing a malformed file returns a best-effort model plus a list of `{ line, message }` errors with 1-based, file-relative line numbers.

## 3. Tasks

### 3.1 Frontmatter

| key | type | notes |
|---|---|---|
| `id` | `PREFIX-n` | required |
| `title` | string | required; the H1 is intentionally not used so Obsidian, the UI and the CLI agree on one title |
| `type` | `task \| bug \| spike \| epic \| subtask` | required; epics are tasks with `type: epic` |
| `status` | string | required; must be one of `config.yml` `statuses` (validated by tooling, not by the parser) |
| `epic` | wikilink | the epic this task belongs to |
| `parent` | wikilink | for `type: subtask`, the parent task |
| `sprint` | string | sprint id from `config.yml` |
| `milestone` | string | milestone id from `config.yml` |
| `assignee`, `reporter` | string | `@handle` for humans, `@claude` etc. for agents |
| `priority` | string | free text, typically `low \| medium \| high \| critical` |
| `labels` | string list | |
| `related` | wikilink list | linked docs, decisions or other tasks (shown on the task page) |
| `created`, `updated` | ISO 8601 string | maintained by tooling |
| `plan_approved_by`, `plan_approved_at` | string, ISO 8601 | the **plan-approval marker** (3.5) |
| anything else | any | **custom fields** — passthrough, exposed on the model as `custom` and validated against `config.yml` `custom_fields` by tooling |

### 3.2 Sections

The body is split on `##` headings. Five section titles are recognised (case-insensitive): `Description`, `Acceptance criteria`, `Implementation plan`, `Decision log`, `Summary`. Any other `##` section is kept verbatim as an unknown section. Deeper headings (`###` …) inside Description / Implementation plan / Summary are ordinary content. Text before the first `##` heading is preserved as a preamble.

- **Description** — markdown, the *why* and scope.
- **Acceptance criteria** — checklist, see 3.3.
- **Implementation plan** — markdown, the *how*; its approval state is the plan-approval marker (3.5).
- **Decision log** — typed entries, see 3.4. Replaces free-form comments.
- **Summary** — markdown, PR-ready completion summary ("Copy for PR").

### 3.3 Acceptance criteria

```text
- [ ] <criterion text>
  - verify: <shell command>
  - evidence: <pass|fail> by <who> at <ISO timestamp> (<commit sha>): <short output>
```

- One top-level checkbox item per criterion; items are numbered by position (1-based) in CLI and UI, never in the file.
- `verify:` is optional. A criterion without `verify:` is **manual**.
- `evidence:` is written by `manatomic tasks task verify` (server/desktop host or agent) — never by the browser/Obsidian. `(<sha>)` and `: <output>` are optional. Output is a single line; long output is truncated by the writer.
- **Writer rules** (`verify.*` in section 6): the command runs with `sh -c` in the parent directory of the data root; output is combined stdout and stderr collapsed to one line and truncated to `max_output` characters keeping the tail (an elided line starts with `…`); a command that outlives `timeout_seconds` is killed and recorded as `fail` with an output starting `timed out after <n>s` — the kill signals the `sh` process, so a still-running grandchild may linger; `(<sha>)` is `git rev-parse --short HEAD` in that directory, omitted outside a git repo; a criterion without `verify:` is manual and is never written.
- The **checkbox is the single source of truth** for "done". Tooling may refuse to check a manual criterion without a human doing it, but the parser only reports state.
- Unknown sub-bullets and continuation lines under an item are preserved.
- **Derived state** (`AcItem.state`):
  1. no `verify:` ⇒ `manual`
  2. evidence result `fail` ⇒ `failing`
  3. evidence result `pass` ⇒ `verified`
  4. otherwise ⇒ `pending`

### 3.4 Decision log

Each entry is a `###` heading of the form `### <type> | <author> | <ISO timestamp>` followed by free markdown until the next entry. `type` is one of `plan`, `decision`, `progress`. `author` is `@handle` (human) or the agent handle (`@claude`). Entries are append-only and kept in file order (chronological). Content under `## Decision log` before the first entry is preserved as a preamble; a `###` heading that does not match the grammar is reported as a parse error and kept verbatim.

```text
### decision | @najdan | 2026-08-22T10:30:00Z
Use the yaml package instead of a hand-written YAML subset.
```

### 3.5 Plan-approval marker

"Approved by you" on the implementation plan is recorded in frontmatter as `plan_approved_by` (who) and `plan_approved_at` (ISO timestamp). Both keys present ⇒ plan approved; tooling removes them whenever the `## Implementation plan` body changes, so approval always refers to the plan as it was read. The approval is *also* a normal `decision` entry in the decision log when a human approves via UI/CLI, which keeps the audit trail readable without the marker.

### 3.6 Example

```markdown file=manatomic/tasks/MAN-12 - Login form validation.md
---
id: MAN-12
title: Login form validation
type: task
status: In Progress
epic: "[[MAN-3 - Onboarding]]"
sprint: S2
milestone: M1
assignee: "@claude"
reporter: "@najdan"
priority: high
labels: [web, forms]
related: ["[[DEC-7 - Client-side validation]]", "[[2026-08-20-login-ux]]"]
component: web
created: 2026-08-20T09:12:00Z
updated: 2026-08-22T10:31:00Z
plan_approved_by: "@najdan"
plan_approved_at: 2026-08-21T15:00:00Z
---

## Description

Users get no feedback when the login form is submitted with an invalid email.
Show inline validation before the request is sent.

## Acceptance criteria

- [x] Invalid email shows an inline error before submit
  - verify: bun test packages/tasks-web -t login-validation
  - evidence: pass by @claude at 2026-08-22T10:30:00Z (a1b2c3d): 4 tests pass
- [ ] Error copy reviewed by design
- [ ] Submit button disabled while the form is invalid
  - verify: bun test packages/tasks-web -t submit-disabled
  - evidence: fail by @claude at 2026-08-22T10:30:00Z (a1b2c3d): expected disabled, got enabled

## Implementation plan

1. Add `validateEmail` to the form model.
2. Render the error under the input.

## Decision log

### plan | @claude | 2026-08-21T14:50:00Z
Proposed the two-step plan above. Validation stays client-side; the API already rejects bad emails.

### decision | @najdan | 2026-08-21T15:00:00Z
Plan approved.

### progress | @claude | 2026-08-22T10:31:00Z
Step 1 done, step 2 in progress. Submit-disabled test still failing.

## Summary

Inline email validation on the login form with disabled submit while invalid.
```

## 4. Docs

Docs live anywhere under `docs/` (sub-folders are free: `brainstorms/`, `research/`, …). Frontmatter:

| key | type | notes |
|---|---|---|
| `title` | string | required |
| `type` | string | free, e.g. `brainstorm`, `research`, `spec` |
| `status` | string | free, e.g. `draft`, `explored`, `final` |
| `tags` | string list | |
| `related` | wikilink list | tasks, decisions or other docs |
| `created`, `updated` | ISO 8601 string | |
| anything else | any | passthrough `custom` |

The body is free markdown and is never restructured.

```markdown file=manatomic/docs/brainstorms/2026-08-20-login-ux.md
---
title: Login UX brainstorm
type: brainstorm
status: explored
tags: [web, onboarding]
related: ["[[MAN-12 - Login form validation]]", "[[DEC-7 - Client-side validation]]"]
created: 2026-08-20
author: "@najdan"
---

# Login UX brainstorm

## Problem

Users abandon the login form when errors only appear after submit.

## Decisions

- Validate inline.
```

## 5. Decisions

One file per decision under `decisions/`, `DEC-<n>` ids. Frontmatter:

| key | type | notes |
|---|---|---|
| `id` | `DEC-n` | required |
| `title` | string | required |
| `status` | `proposed \| accepted \| rejected \| superseded` | required |
| `date` | ISO 8601 date string | required |
| `deciders` | string list | |
| `supersedes`, `superseded_by` | wikilink | |
| `related` | wikilink list | |
| anything else | any | passthrough `custom` |

The body is free markdown; the conventional headings are `## Context`, `## Decision`, `## Consequences` but they are not enforced.

```markdown file=manatomic/decisions/DEC-7 - Client-side validation.md
---
id: DEC-7
title: Client-side validation
status: accepted
date: 2026-08-21
deciders: ["@najdan"]
related: ["[[MAN-12 - Login form validation]]"]
---

## Context

The API rejects malformed emails but the form gives no feedback.

## Decision

Validate in the browser before submitting; keep server validation as the safety net.

## Consequences

Duplicate rules live in two places until a shared schema exists.
```

## 6. `config.yml`

```yaml file=manatomic/config.yml
prefix: MAN
statuses:
  - To Do
  - In Progress
  - name: In Review
    category: in_progress
  - name: Done
    category: done
issue_types: [task, bug, spike, epic, subtask]
custom_fields:
  - name: component
    type: select
    options: [web, server, core]
  - name: estimate
    type: number
sprints:
  - id: S1
    name: Sprint 1
    start: 2026-08-11
    end: 2026-08-22
  - id: S2
    name: Sprint 2
    start: 2026-08-25
    end: 2026-09-05
    goal: Ship login
milestones:
  - id: M1
    name: v1 public beta
    start: 2026-09-01
    due: 2026-10-01
definition_of_done:
  - Tests pass
  - Docs updated
verify:
  enabled: true
  auto_check: false
  timeout_seconds: 60
  max_output: 200
```

- `prefix` (required) — task id prefix.
- `statuses` — ordered list; each entry is a string or `{ name, category }` where `category` is `todo | in_progress | done`. Strings default to `todo` for the first entry, `done` for the last and `in_progress` in between. The first status is the default for new tasks.
- `issue_types` — allowed `type` values (default `[task, bug, spike, epic, subtask]`).
- `custom_fields` — `{ name, type: text | number | select | date | boolean, options?, required? }`.
- `sprints` — `{ id, name, start, end, goal? }`; a task is in a sprint when its `sprint` equals the id.
- `milestones` — `{ id, name, start?, due?, description? }`; a task is in a milestone when its
  `milestone` equals the id. Both dates are ISO `YYYY-MM-DD` and both are optional: `due` is what
  puts a milestone on the roadmap's chart, and a missing `start` is derived by the UI from the
  previous milestone's `due` — derived, never stored.
- `definition_of_done` — default checklist copied into new tasks' acceptance criteria by tooling.
- `verify` — how `manatomic tasks task verify` may run acceptance-criteria commands: `enabled` (default `false`), `auto_check` (default `false`), `timeout_seconds` (positive integer, default `120`) and `max_output` (positive integer, default `200`, the characters of output kept in an evidence line). The whole map is optional; a key of the wrong type is a config error naming it and keeps that key's default.

Unknown top-level keys are preserved and exposed as `custom`.

A UI save (`setConfig`, section 8) rewrites only the keys it was given and edits the file
through the YAML document model, so unknown top-level keys, untouched managed keys and the
comments attached to either survive byte-for-byte — a comment *inside* a list the save
rewrites does not, because that list is re-emitted. A status is written as a bare string
when its category is the one its position already implies and as `{ name, category }`
otherwise, so a hand-written file does not churn. `verify` is written one sub-key at a time,
leaving sub-keys the UI does not manage alone.

## 7. Concurrency (v1): last-write-wins with a conflict signal

Three writers touch the same files: the human's editor/Obsidian, agents via the CLI, and the UI via the server. v1 does **not** merge.

1. Every read returns the parsed model **and** a `fingerprint` (`fingerprint(text)`, FNV-1a over the exact bytes) of the text it was parsed from.
2. A writer passes the fingerprint it read back with the write (`If-Match` semantics). The server re-reads the file, compares fingerprints, writes the new content regardless (**last write wins**), and reports `conflict: true` when the fingerprints differed. The CLI prints a warning; the UI shows a conflict banner with a "reload" action.

   Over HTTP the tasks server realises this literally: the client sends the fingerprint it read in the **`If-Match` request header** (the bare fingerprint, not ETag-quoted), and every mutation answers `{ entry, fingerprint, conflict }` — **`conflict`** being that flag. Create and delete endpoints have nothing to match against and ignore the header. The endpoint list is in the README's Server section.

   The CLI realises it with the **`--if-match <fingerprint>`** option on its task-mutating commands (`task edit`, `task verify`): pass the fingerprint a `task view` (or a previous mutation) printed, and the CLI compares it against the file it loads for this invocation, writes regardless, and on a mismatch prints a conflict warning to stderr in plain mode; with `--json` the mutation payload carries the same **`conflict`** flag instead. Without `--if-match` there is nothing to match against — no warning, `conflict: false` — and create commands take no fingerprint, exactly like the server's create endpoints.
3. The file watcher re-parses on change and pushes the new model; an open UI page that has unsaved field edits shows the same banner instead of silently reloading.
4. Because tools re-render only the node they change and preserve everything else byte-for-byte, two non-overlapping edits that race lose at most the later one's view of the earlier one — never unrelated content.

Proper three-way merge is out of scope for v1 (tracked as a spike).

## 8. Repository semantics

`packages/tasks-core` exposes a `Repository` (`openRepository(fs, { root })`) that is the one engine the CLI, the server and editor hosts use to read and change a data root. It talks to files only through an injected async `FileSystemAdapter` (`readFile`, `writeFile`, `deleteFile`, `exists`, recursive `listFiles`); `MemoryFileSystem` is the in-memory implementation used by tests and browser demos. Loading builds an in-memory index; every mutation is validated against `config.yml` and the index **before** anything is written, re-renders only the touched node (section 2) and returns the new fingerprint (section 7).

- **Id allocation** — the next task id is `<PREFIX>-<max existing n + 1>` and the next decision id is `DEC-<max existing n + 1>`. Gaps are never refilled and a deleted id is never reused within a session: the allocator remembers the highest number it has seen even after the file is gone. Loading fails with a typed error when `config.yml` is missing or has no `prefix`.
- **New tasks** — `createTask` writes `tasks/<ID> - <title>.md` (the filename title is the task title with `/ \ : ? * " < > |` stripped), with `type` (default `task`), the first configured status unless one is given, `created` and `updated` set to today, an optional Description, and every `definition_of_done` entry seeded as an unchecked acceptance criterion. `createDecision` writes `decisions/DEC-<n> - <title>.md` with `status` (default `proposed`) and today's `date`.
- **New docs** — `createDoc({ title, dir?, type?, body? })` writes `docs/<dir>/<title>.md` (filename title sanitized like task filenames) with frontmatter `title`, `type` (only when given) and `created` set to today, followed by the optional markdown body. The new doc is registered in the index before the rebuild, so wikilinks to it resolve immediately. It is rejected when the key is already taken or the file exists (`duplicate-id`), when the title has no usable filename characters or when `dir` contains a `..` segment (`invalid-path`).
- **Recording evidence** — `setAcceptanceEvidence(id, n, evidence, { checked? })` writes the `evidence:` sub-bullet of criterion `n` (1-based), checking the box in the same write when `checked` is given, so evidence and checkbox never disagree on disk. A verify run is one write per criterion: interrupt it (or delete the task, or drop a criterion from disk mid-run, which makes the next write throw `unknown-id` / `ac-out-of-range`) and the evidence already written stays.
- **Adding acceptance criteria** — `addAcceptanceCriterion(id, text, verify?)` appends one unchecked criterion after the existing ones (creating the `## Acceptance criteria` section when missing), with a `verify:` sub-bullet only when a command is given — so a criterion added without one is `manual` (section 3.3).
- **Doc identity** — a doc has no id; it is identified by its path relative to `docs/` without the `.md` extension (`brainstorms/2026-08-20-login-ux`). A wikilink without a leading id is resolved against docs: first by exact key, then by basename when exactly one doc has that basename. An ambiguous basename or a missing target is an index warning, not an error, and yields no back-link.
- **Referenced-by** — every task, doc and decision carries `referencedBy: { from: { kind, id }, field }[]`, derived from task `epic`, `parent` and `related` links and decision `supersedes` and `related` links. Subtasks whose `parent` resolves appear in the parent's `children`; tasks with `type: epic` are listed as `epics`. Links are re-derived from the in-memory models after every mutation, without re-reading files.
- **Validation** — `status` must be a configured status; `type` must be in `issue_types`; `sprint` / `milestone` must be configured ids; `epic` must link to an existing epic, `parent` to an existing task (never itself), `related` entries must resolve; custom fields are checked against `custom_fields` (unknown key, type and `select` options; `required` fields must be present on create). A rejected mutation throws a `RepositoryError` (`code`, optional `field` / `id`) and leaves file bytes and index untouched. Task mutations set `updated` to today; decision mutations never touch `date`.
- **Delete** — `deleteTask` / `deleteDecision` remove the file through the adapter. There is no archive folder: git history is the archive. Links that pointed at the deleted entry become index warnings.
- **Editing `config.yml`** — `setConfig(patch)` rewrites the managed sections (`statuses`, `issueTypes`, `customFields`, `definitionOfDone`, `verify`) in place, merging into whatever is on disk at write time (section 6); `prefix` is not patchable, since changing it would leave existing ids on the old prefix and restart numbering. The patch is validated first (non-empty unique status names with a known category, unique non-empty issue types, unique custom-field names with a known type and a `select` carrying at least one option, positive `verify` numbers), and then checked against the index: a status, issue type or custom-field name the patch drops — a rename counts as a drop — that tasks still carry is refused with `config-in-use` naming the affected task ids in id order. A name already absent from the current config was not removed by this edit, so a pre-existing orphan never blocks a save. Nothing is written unless the result reparses cleanly; a successful save replaces the loaded config and rebuilds the index, so the next mutation is validated against the new one.
- **Issue types** — for now the model only knows the five built-in `type` values (`task | bug | spike | epic | subtask`); an `issue_types` entry outside that list is accepted in `config.yml` but rejected on tasks until the model grows custom issue types (tracked as a follow-up).
