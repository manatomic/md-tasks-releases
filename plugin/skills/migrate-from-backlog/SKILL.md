---
name: migrate-from-backlog
description: One-shot migration of a backlog.md data root (a backlog/ directory) to a manatomic md-tasks data root. Use when the user asks to migrate from backlog.md, convert backlog/ to manatomic tasks, or adopt md-tasks in a repo that still has a backlog/ directory. An agent procedure — read the backlog files directly, write through the manatomic tasks CLI, verify the result, and only then remove backlog/.
---

# Migrate a backlog.md root to manatomic tasks

A phased agent procedure encoding the mapping that migrated md-tasks' own backlog (its DEC-1 decision record). It is deliberately a procedure, not a `tasks migrate` command: migration is one-shot per repo. Read `backlog/` files directly; write only through `manatomic tasks` (the `cli` skill is the command reference), except the three **migration-scoped direct edits** defined below. The destructive step — removing `backlog/` — is last and gated behind verification.

Ground rules:

- Migrate on a **clean git tree**, so the whole migration is one reviewable commit. Git history keeps the old tree; nothing is archived.
- Never fabricate data: no invented `evidence:` lines, no invented timezones or seconds on timestamps, no invented section content.
- Anything in the source with no manatomic equivalent (drafts, unfamiliar frontmatter keys, milestone/sprint-like features): surface it to the user and decide together — never drop silently.

## Phase 0 — inventory and id map

1. Read every `backlog/tasks/*.md` frontmatter (`id`, `title`, `status`, `parent_task_id`, `ordinal`, `labels`, `dependencies`, `documentation`, `assignee`, `priority`, `created_date`, `updated_date`) and `backlog/config.yml` (`statuses`). List `backlog/docs/**` and `backlog/decisions/**` (if present).
2. Build the id table. manatomic ids are `PREFIX-n` with no dotted children: each task whose id others name as `parent_task_id` becomes an **epic** and takes the next id first; its children follow in `ordinal` order; remaining tasks follow in id order. Dotted ids (`TASK-1.10`) sort numerically, not lexically.
3. Count what Phase 5 must re-find: tasks; per task the AC total and checked total, and the `## Comments` entry total; docs; decisions.
4. Ask the user for the id prefix (letters and digits, starting with a letter) and show the full `old id → new id` table for confirmation **before writing anything**.

## Phase 1 — scaffold the data root

`manatomic tasks init --prefix <P>` (it refuses a folder that already has a `config.yml` — if a data root exists, stop and ask). When `backlog/config.yml` declares statuses other than the default `To Do / In Progress / Done`, edit `manatomic/config.yml`'s `statuses` to match **before creating tasks** — status values are validated on every write.

## Phase 2 — tasks, in id-table order

Create in table order and check that each `task create` prints the id the table promised. Per task:

| backlog.md source | manatomic target | how |
|---|---|---|
| `id` | new `PREFIX-n` | allocation is ordinal — creation order realizes the table |
| `title` | `title` | `task create "<title>"` |
| the parent of others | `type: epic` | `--type epic` on create; drop a literal `epic` label if present |
| `status` | `status` | `--status "<mapped>"` |
| `assignee` (a list) | `assignee` (one handle) | first entry, `--assignee` |
| `labels` | `labels` | `--labels a,b` |
| `parent_task_id` | child's `epic:` wikilink | `--set epic=<new epic id>` — children are epic members, not subtasks, so epic progress counts them |
| `dependencies` + `documentation` | `related` wikilinks | **deferred to the link pass (Phase 3b)** — `related` targets must resolve at write time, and forward task references and doc keys do not exist yet |
| `priority` | `priority` | `--set priority=` |
| `ordinal` | — | dropped (position ordered only the backlog list) |
| `created_date` (`YYYY-MM-DD HH:MM`) | `created` | a follow-up `task edit --set "created=YYYY-MM-DDTHH:MM"` — on the **create** call the today-stamp wins over `--set created=`, on an edit the explicit value sticks; keep minute precision, `T` separator, no invented timezone or seconds |
| `updated_date` | `updated` | **cannot be set via CLI** — every edit re-stamps `updated` to today; restore it in the historic-metadata pass below, which is why that pass must be the task's very last write |
| `## Description` | `## Description` | `--description "<text>"` on create, `<!-- SECTION:* -->` markers stripped |
| `## Acceptance Criteria` | AC checklist | per criterion, one edit call: `--ac-add "<text>" [--ac-verify "<cmd>"]`; drop the `#n` prefix (position numbers them); a `(verify: <cmd>)` suffix becomes the `--ac-verify` command; `(verify: manual — <steps>)` is folded into the criterion text as `(manual: <steps>)` with **no** `--ac-verify`, which is what makes it manual; `<!-- AC:* -->` markers stripped |
| checked boxes | checked boxes | `--check-ac <n>` after all criteria are added; never write `evidence:` — backlog recorded none and evidence only ever comes from a real verify run |
| `## Implementation Plan` | `## Implementation plan` | `--plan -` with the body on stdin, markers stripped |
| `## Comments` entries | `## Decision log` typed entries | per entry, in order: `--log <type> --author <original author> --message "<body>"` — `decision` when the body reads `Q: … / Recommendation: … / User chose: …` (or otherwise records a choice), `plan` for plan proposals, `progress` for the rest; `<!-- COMMENTS:* -->` markers and the `---` delimiters are source syntax, not content |
| `## Final Summary` | `## Summary` | `--summary "<text>"` |
| `## Implementation Notes`, any unrecognised section | preserved verbatim | no CLI path — historic-metadata pass below |

**The historic-metadata pass** — the migration-scoped exception to the tasks-are-CLI-only rule, run once per task *after every CLI write for it* (including the Phase 3b link pass and the `created` fix — any later edit re-stamps `updated` to today and undoes step 2), then immediately re-parsed:

1. In the `### <type> | <author> | <timestamp>` decision-log headings, replace the CLI-stamped timestamps with the original comment `created` values (`2026-09-02 13:37` → `2026-09-02T13:37`), keeping entry order.
2. Set `updated:` to the source `updated_date` (same format).
3. Append unrecognised sections (`## Implementation Notes`, …) **verbatim** — the format preserves unknown sections byte-for-byte.

After the pass, `manatomic tasks task view <id>` must exit 0 with the expected fields — a parse error means the edit broke the file: fix before moving on. These three writes are sanctioned only during migration; afterwards the CLI-only rule is absolute.

## Phase 3 — docs and decisions

- Docs keep their layout: `doc create "<name>" --dir <subfolder> [--type <type>]`, then write the body by editing the created file — doc bodies are the format's sanctioned file-edit surface (see the `cli` skill's rule split). Strip nothing: doc bodies move verbatim.
- Decisions (when the source has them): `decision create "<title>" --status <status> --deciders <a,b>`, then edit the body in place. Ids renumber to `DEC-<n>` in creation order — add them to the id table.

### Phase 3b — link pass

Now that every task, doc and decision exists, walk the tasks again and set `related`: `task edit <id> --set related=<id,id,dockey>` (bare ids and doc keys are wikilinked automatically; use the *new* ids from the table). The format has no dependency edge; `related` is its "see also". A target that still does not resolve is a mapping mistake — fix the map, don't drop the link.

## Phase 4 — prose rewrite

Rewrite prose **only where it names a moved thing**: old task ids (`TASK-1.10`) become the mapped ids, and `backlog/docs/…` paths become the new doc keys/paths. Bare historic references ("1.11 shipped this") stay untouched — the recorded id table decodes them. Task sections rewrite through the CLI (`--description -`, `--plan -`, `--summary`); doc bodies are file edits.

## Phase 5 — verify (the gate)

All of it, before anything is deleted:

1. `task list --plain`: count equals the Phase 0 task count; ids match the table.
2. Per task: `task view <id>` exits 0; AC total and checked total match the source; decision-log entry count matches the source comment count; every criterion that had `(verify: …)` shows it and none gained evidence.
3. Per epic: child count (`epic` backlinks) matches the source `parent_task_id` fan-out.
4. `doc list` / `decision list` counts match; spot-open one migrated doc.
5. Anything the repo's own tooling checks (tests reading doc paths, CI globs referencing `backlog/`): grep the repo for `backlog/` references and fix them now.

Any mismatch: fix and re-verify. **Do not proceed while anything is red.**

## Phase 6 — remove and record

1. `git rm -r backlog/`.
2. Record the migration as a decision: `decision create "Migrate the project backlog from backlog.md to manatomic tasks" --status accepted --deciders <user>`, body carrying the context, the full old→new id table, and the mapping notes (what was kept, moved, dropped) — md-tasks' own DEC-1 is the model.
3. Commit the whole migration as one commit.
