---
name: cli
description: Reference for the manatomic tasks CLI (md-tasks) — the markdown task system where tasks, docs and decisions live as files in a data root. Use whenever working in a repo with a manatomic data root (a manatomic/ directory holding config.yml, tasks/, docs/, decisions/), or when the user mentions manatomic, md-tasks, or asks to create, edit, list or verify tasks, docs or decision records managed by it.
---

# manatomic tasks CLI reference

The `manatomic tasks` CLI manages tasks, docs and decision records as markdown files in a data root (default `./manatomic`) committed with the code. Every command is non-interactive: output on stdout, one-line failures on stderr, exit codes a script can branch on (`0` success, `1` failure, `2` usage error). Nothing is written to stdout when a command fails.

```
manatomic tasks <command> [arguments] [options]
```

Global options, accepted by every command:

| option | meaning |
|---|---|
| `--root <dir>` | data root, relative to cwd; absolute paths used unchanged (default `./manatomic`) |
| `--plain` | plain text output — the default, never colored |
| `--json` | JSON output on stdout, `{"error":{"code","message","field?","id?"}}` on stderr |
| `--help` | usage for the tool or for one command |

## The rule split: tasks via CLI, doc bodies as files

- **Tasks and decisions are CLI-only. Never hand-edit their files.** Every mutation goes through the CLI, which validates against `config.yml`, re-renders only what it changes, maintains `updated`, and prints the new fingerprint. Hand edits race other writers and skip validation.
- **Doc bodies are legitimately edited as files.** `doc create` only scaffolds frontmatter (`title`, `created`, optional `type`); the body is written and revised by editing the file directly — there is no `doc update` command, by design.

## Reading

| action | command |
|---|---|
| List tasks | `manatomic tasks task list` — one line per task: `ID  status  type  title` |
| Filter tasks | `task list --status "To Do" --type bug --epic MAN-2 --sprint S1 --milestone M1 --assignee @claude --label web` (AND-combined) |
| View a task | `manatomic tasks task view MAN-1` — fields, numbered AC, decision log |
| List / view docs | `manatomic tasks doc list`, `doc view <key>` (key = path under `docs/` without `.md`, e.g. `brainstorms/2026-08-20-login-ux`) |
| List / view decisions | `manatomic tasks decision list`, `decision view DEC-1` |
| Search everything | `manatomic tasks search "term"` — case-insensitive substring over ids, titles and body text of tasks, docs and decisions; prints a `tasks:` / `docs:` / `decisions:` header per non-empty kind with the same rows as the list commands |
| Project config | `manatomic tasks config show` |
| Structured output | append `--json` to any of the above |

**Start sessions with `config show`**: it reveals the configured statuses (with their `todo`/`in_progress`/`done` categories — the first status is the default for new tasks), issue types, sprints, milestones, custom fields, the `definition_of_done` checklist seeded into new tasks' AC, and the `verify` settings.

## Creating

| action | command |
|---|---|
| Scaffold a data root | `manatomic tasks init --prefix MAN` (creates `config.yml`, `tasks/`, `docs/`, `decisions/`; refuses a folder that already has a `config.yml`) |
| Create a task | `manatomic tasks task create "Title" --description "why + scope" --labels a,b --set priority=high` — prints the new id |
| Create a doc | `manatomic tasks doc create "Title" --dir brainstorms --type brainstorm` — prints the key; write the body by editing the file |
| Create a decision | `manatomic tasks decision create "Statement" --status accepted --deciders @najdan` — prints the next `DEC-<n>` |

`task create` also takes `--type`, `--status`, `--epic <id>`, `--parent <id>`, `--sprint <id>`, `--milestone <id>`, `--assignee <handle>`, and `--set <key=value>` (repeatable, same syntax as `task edit`). New tasks get every `definition_of_done` entry from `config.yml` seeded as an unchecked acceptance criterion.

Acceptance criteria at create time: `--ac "criterion"` is repeatable and appends unchecked criteria in flag order, after the seeded `definition_of_done`. Each `--ac-verify "cmd"` attaches to the nearest preceding `--ac` (at most one per criterion; omit it for a manual criterion): `task create "Signup" --ac "Form renders" --ac-verify "bun test signup" --ac "Reviewed by design"`.

## Editing tasks

All through `manatomic tasks task edit <id>`; a successful mutation prints `<id>  <fingerprint>`.

| action | command |
|---|---|
| Status / assignee / any frontmatter key | `task edit MAN-1 --set "status=In Progress" --set assignee=@claude` |
| Remove a frontmatter key | `--set key=` (empty value) |
| Add acceptance criteria | `--ac-add "Outcome-focused criterion" --ac-verify "bun test login"` (repeatable pairs, appended in flag order; each `--ac-verify` attaches to the nearest preceding `--ac-add` — omit it for a manual criterion) |
| Check / uncheck AC n (1-based) | `--check-ac 1 --uncheck-ac 2` (repeatable) |
| Remove AC n (1-based) | `--remove-ac 2` (repeatable; the rest renumber by position — re-`view` before citing numbers again) |
| Append a decision-log entry | `--log decision --author @claude --message "text"` (`--log plan\|decision\|progress`) |
| Replace the Summary | `--summary "PR-ready completion summary"` |
| Append to the Summary | `--append-summary "paragraph"` (repeatable — each use appends one paragraph, without resending the existing text; creates the section if missing; combined with `--summary` in one call, the replace applies first) |
| Replace Description / Implementation plan | `--description <file>` / `--plan <file>`, or `-` for stdin (stdin usable once per call) |
| Guard against concurrent writers | `--if-match <fingerprint>` — the fingerprint from your read; a mismatch warns but the write still wins (see Fingerprints and write conflicts) |

`--set` semantics: `labels` and `related` split on commas; `epic`, `parent` and each `related` entry accept a bare id and are stored as wikilinks (`--set related=MAN-5,DEC-1`); custom fields are coerced to their configured `number`/`boolean` type. Statuses, types, sprints, milestones and custom fields are validated against `config.yml` — a rejected value is refused without being written. But flags in one call apply **in order, each writing immediately**: when a later flag is rejected (exit 1), earlier flags from the same call are already on disk. After a failed multi-flag edit, `task view` to see what landed.

Multi-line content: `--description` and `--plan` take a **file path or stdin**, not inline text (only `task create --description` is inline). Write the markdown to a temp file, or pipe it: `manatomic tasks task edit MAN-1 --plan -` with the plan on stdin.

## Local web UI

`manatomic tasks serve [--port <n>] [--host <addr>]` serves the data root over HTTP and WebSocket until interrupted — a browser UI plus JSON API for humans reviewing what agents did (default `http://127.0.0.1:4400`; `--port 0` picks a free port). Agents normally don't need it; prefer the CLI.

## Field semantics

- **Description** — the *why*: purpose and scope, no implementation detail.
- **Acceptance criteria** — the *what*: outcome-focused checkboxes, numbered 1-based by position in CLI/UI output (never in the file). Each criterion may carry a `verify:` sub-bullet with a shell command; **a criterion without `verify:` is manual**. The checkbox is the single source of truth for "done".
- **Implementation plan** — the *how*; its approval state is the plan-approval marker (below).
- **Decision log** — append-only typed entries (`plan | decision | progress`, each with author and timestamp). This replaces free-form comments: questions asked, recommendations, choices made and progress notes all go here via `--log`.
- **Summary** — short PR-ready completion summary.

## Verify evidence

`manatomic tasks task verify MAN-1 --by @claude` runs each criterion's `verify:` command (`--ac <n>` limits to one) and records the outcome as an `evidence:` sub-bullet — `pass|fail`, who, when, the commit sha, and one line of output. Failing commands are evidence, not errors: the criterion reads `failing`, its checkbox is untouched. A pass checks the box only when `verify.auto_check` is set; otherwise check it separately with `--check-ac <n>`. Manual criteria are never run.

- Verify is **opt-in**: without `verify.enabled: true` in `config.yml` the CLI refuses with `verify-disabled`. `config show` reveals the setting (plus `timeout_seconds` and `max_output`).
- Commands run with `sh -c` in the **parent directory of the data root** (`<repo>/manatomic` → `<repo>`), so write them repo-relative.
- Prefer recorded evidence over trusting checkboxes: `task view --json` reports each criterion's derived `state` — `manual`, `pending` (no evidence yet), `verified` or `failing`. Plain `task view` prints only the checkbox and verify command; evidence sub-bullets live in the file.

## Plan approval

Frontmatter keys `plan_approved_by` + `plan_approved_at` (both present ⇒ plan approved) record who approved the implementation plan and when. **Replacing the plan body clears the marker automatically**, so approval always refers to the plan as it was read. To record an approval the user gave:

```sh
manatomic tasks task edit MAN-1 --set plan_approved_by=@najdan --set "plan_approved_at=2026-09-20T15:00:00Z"
manatomic tasks task edit MAN-1 --log decision --author @najdan --message "Plan approved."
```

`task view --json` reports it as `planApproval`. Check it before implementing: a task with a plan but no marker is waiting on a human.

## Fingerprints and write conflicts

Every read returns a `fingerprint` of the file's exact bytes (`task view` prints it as a field) and every mutation prints the new one (`<id>  <fingerprint>`). Writes are **last write wins** — the CLI never blocks a write — but the task-mutating commands (`task edit`, `task verify`) take `--if-match <fingerprint>`: pass the fingerprint from your last `task view` or mutation, and when the file changed under you the CLI still writes and tells you — a `conflict:` warning on stderr in plain mode, a `"conflict": true` field in the `--json` payload (stderr stays reserved for errors there). Without `--if-match` there is no check and no warning. When other writers may be active (the web UI, an editor, another agent), always pass it, and re-read before editing further when it flags a conflict; the web UI gets the same signal over HTTP (`If-Match` header). Hand-editing task files is forbidden for the same reason: it races the CLI/UI writers and skips validation.

## Known gaps and workarounds

- **Repeated `--remove-ac` numbers all refer to the pre-removal numbering** (`--remove-ac 1 --remove-ac 3` removes the original #1 and #3), removal runs after `--check-ac`/`--uncheck-ac` in the same call, and one out-of-range number fails the call before anything is removed.
- **No AC reword** via the CLI — `--remove-ac` then `--ac-add` re-adds it at the end (losing its position), so for a small wording fix prefer noting it in the decision log.
- **No archive/delete commands** — git history is the archive.
- **No `doc update`** — deliberate: edit doc bodies as files (see the rule split above).

## If `manatomic` is not installed

```sh
curl -fsSL https://github.com/manatomic/md-tasks-releases/releases/latest/download/install.sh | bash
```

Installs a single-file binary as `~/.local/bin/manatomic` (`MANATOMIC_VERSION=v1.2.3` pins a tag, `MANATOMIC_INSTALL_DIR` another directory). The file format spec ships in the same repo under `docs/manatomic-file-format.md`.
