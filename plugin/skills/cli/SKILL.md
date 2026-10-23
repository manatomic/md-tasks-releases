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
| `--root <dir>` | data root, relative to the current directory; absolute paths used unchanged (default `./manatomic`) |
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
| Filter tasks | `task list --status "To Do" --type bug --epic MAN-2 --sprint S1 --milestone M1 --assignee @claude --label web` (AND-combined; `--epic` matches direct members only — tasks whose `epic:` is set to it; `task view <epic>` shows the full tree). Archived tasks are left out: `--archived` lists only them, `--include-archived` adds them |
| View a task | `manatomic tasks task view MAN-1` — fields, a `## Subtasks (<done>/<total> done)` tree of everything under the task (an epic's members, any task's `parent:`-linked subtasks, recursively, indented two spaces per level; omitted when empty), numbered AC — a criterion's steps, the lines under it in the file, print indented under its text — and the decision log; `--json` exposes each criterion's steps as `steps: string[]` (common leading indentation removed, relative indentation and numbering kept, `[]` when there are none) and adds `subtasks: [{ id, title, type, status, subtasks }]` (`[]` when empty) |
| List / view docs | `manatomic tasks doc list`, `doc view <key>` (key = path under `docs/` without `.md`, e.g. `brainstorms/2026-08-20-login-ux`) |
| List / view decisions | `manatomic tasks decision list`, `decision view DEC-1` |
| Search everything | `manatomic tasks search "term"` — case-insensitive substring over ids, titles and body text of tasks, docs and decisions; prints a `tasks:` / `docs:` / `decisions:` header per non-empty kind with the same rows as the list commands; archived tasks only with `--include-archived` |
| Project config | `manatomic tasks config show` |
| Project avatar | `manatomic tasks config avatar [text] [--color <hex>] [--image <path>]` / `config avatar --clear` — see below |
| Structured output | append `--json` to any of the above |

**Start sessions with `config show`**: it reveals the configured statuses (with their `todo`/`in_progress`/`done` categories — the first status is the default for new tasks), issue types, sprints, milestones, custom fields, the `definition_of_done` checklist seeded into new tasks' AC, the `verify` settings, the `features:` switches, `archive: after_days`, and the project's `avatar` when one is set.

**Feature switches** — `features:` in `config.yml`: `sprints`, `epics`, `roadmap`, `archive`, `docs`, `agents` and `plan_approval`, each a boolean defaulting to `true` — say what the project uses. They only shape the web app (a switched-off feature's screen, nav item, fields, filters and keys are hidden); **no CLI command is refused because a feature is off**. With `plan_approval: false` an unapproved plan does not wait on anyone. A non-boolean value is a config error naming the key (the default stays); unknown keys are kept and ignored:

```yaml
features:
  sprints: false
  plan_approval: false
archive:
  after_days: 30   # task archive --stale and the web's Archive now use it
```

**Project avatar** — how the project shows in the hub's rail and Projects page, stored under `avatar:` in `config.yml`: `config avatar [text] [--color <hex>] [--image <path>]` sets only the parts given (text 1–4 characters, an emoji counts as one; color `#rgb`/`#rrggbb`; image an existing png/jpg/gif/webp/svg file inside the repo folder — the data root's parent — resolved from the cwd and stored relative to the repo folder). An empty value (`--color ""`) removes that part, `--clear` the whole avatar; it prints the resulting `avatar:` block (`avatar: none` when cleared; `--json`: `{"avatar": {...} | null}`). A bad part or an image outside the repo exits 1 (`config-invalid`), nothing written; no part at all, or `--clear` with a part, exits 2. Pointing the avatar elsewhere deletes an image uploaded from the web UI's Settings (`<data root>/avatar-<hash>.<ext>`), never another file. A running hub shows the change without a restart.

## Creating

| action | command |
|---|---|
| Scaffold a data root | `manatomic tasks init --prefix MAN` (creates `config.yml`, `tasks/`, `docs/`, `decisions/`; refuses a folder that already has a `config.yml`; also records the new root in the hub registry when `hub.yml` already exists — never creates it, and a registry failure only warns on stderr) |
| Create a task | `manatomic tasks task create "Title" --description "why + scope" --labels a,b --set priority=high` — prints the new id |
| Create a doc | `manatomic tasks doc create "Title" --dir brainstorms --type brainstorm` — prints the key; write the body by editing the file |
| Create a decision | `manatomic tasks decision create "Statement" --status accepted --deciders @najdan` — prints the next `DEC-<n>` |

`task create` also takes `--type`, `--status`, `--epic <id>`, `--parent <id>`, `--sprint <id>`, `--milestone <id>`, `--assignee <handle>`, and `--set <key=value>` (repeatable, same syntax as `task edit`). New tasks get every `definition_of_done` entry from `config.yml` seeded as an unchecked acceptance criterion.

Acceptance criteria at create time: `--ac "criterion"` is repeatable and appends unchecked criteria in flag order, after the seeded `definition_of_done`. Each `--ac-verify "cmd"` attaches to the nearest preceding `--ac` (at most one per criterion; omit it for a manual criterion): `task create "Signup" --ac "Form renders" --ac-verify "bun test signup" --ac "Reviewed by design"`.

A manual criterion with steps is **one `--ac` value with real newlines inside the quotes** (no `\n` escapes): the first line is the criterion, then each step on its own line numbered `1.`, `2.`, …, a detail as a `-` bullet under its step. The CLI indents the steps two spaces under the checkbox line, keeping their relative indentation, whether or not you indented them:

```sh
manatomic tasks task create "Checkout" --ac "Manual — walk the happy path
1. Open the cart with one item
2. Pay with the test card
   - the confirmation page names the order"
```

lands in the task file as

```md
- [ ] Manual — walk the happy path
  1. Open the cart with one item
  2. Pay with the test card
     - the confirmation page names the order
```

`task view` prints the steps under the criterion and `--json` lists them as `steps`, so whoever walks the criterion sees them.

## Editing tasks

All through `manatomic tasks task edit <id>`; a successful mutation prints `<id>  <fingerprint>`.

| action | command |
|---|---|
| Status / assignee / any frontmatter key | `task edit MAN-1 --set "status=In Progress" --set assignee=@claude` |
| Remove a frontmatter key | `--set key=` (empty value) |
| Add acceptance criteria | `--ac-add "Outcome-focused criterion" --ac-verify "bun test login"` (repeatable pairs, appended in flag order; each `--ac-verify` attaches to the nearest preceding `--ac-add` — omit it for a manual criterion; a multi-line value writes the criterion with its steps indented under it, exactly as `task create --ac` does) |
| Check / uncheck AC n (1-based) | `--check-ac 1 --uncheck-ac 2` (repeatable) |
| Remove AC n (1-based) | `--remove-ac 2` (repeatable; the rest renumber by position — re-`view` before citing numbers again) |
| Reword AC n | `--edit-ac 2="New wording"` (repeatable; extra lines in the value become its steps, a one-line value keeps the steps it had; keeps the checkbox and evidence) |
| Set / remove AC n's verify command | `--set-ac-verify 2="bun test login"` / `--set-ac-verify 2=` (repeatable; either clears that criterion's evidence and unticks it) |
| Move AC n | `--move-ac 3:1` (to position 1; once per call, not with `--remove-ac`; keeps checkbox and evidence) |
| Append a decision-log entry | `--log decision --author @claude --message "text"` (`--log plan\|decision\|progress`) |
| Replace the Summary | `--summary "PR-ready completion summary"` |
| Append to the Summary | `--append-summary "paragraph"` (repeatable — each use appends one paragraph, without resending the existing text; creates the section if missing; combined with `--summary` in one call, the replace applies first) |
| Replace Description / Implementation plan | `--description <file>` / `--plan <file>`, or `-` for stdin (stdin usable once per call) |
| Guard against concurrent writers | `--if-match <fingerprint>` — the fingerprint from your read; a mismatch warns but the write still wins (see Fingerprints and write conflicts) |

Numbers in `--edit-ac`, `--set-ac-verify`, `--move-ac`, `--check-ac` and `--remove-ac` all refer to the numbering **before** the call.

`--set` semantics: `labels` and `related` split on commas; `epic`, `parent` and each `related` entry accept a bare id and are stored as wikilinks (`--set related=MAN-5,DEC-1`); custom fields are coerced to their configured `number`/`boolean` type. Statuses, types, sprints, milestones and custom fields are validated against `config.yml` — a rejected value is refused without being written. But flags in one call apply **in order, each writing immediately**: when a later flag is rejected (exit 1), earlier flags from the same call are already on disk. After a failed multi-flag edit, `task view` to see what landed.

Multi-line content: `--description` and `--plan` take a **file path or stdin**, not inline text (only `task create --description` is inline). Write the markdown to a temp file, or pipe it: `manatomic tasks task edit MAN-1 --plan -` with the plan on stdin.

## Archiving tasks

`manatomic tasks task archive MAN-4 MAN-5 --by @claude` moves tasks to `tasks/archive/` (keeping any sub-folder) and stamps `archived_at` and `archived_by`; `task restore MAN-4` moves one back and drops both. Bulk: `task archive --done-before <days> --by <handle>` archives every Done task last updated more than `<days>` days ago, and `task archive --stale --by <handle>` does the same with `archive.after_days` (default 30) — give exactly one of ids, `--done-before` or `--stale`. Archive prints one archived id per line (`--json`: `{"archived": [...]}`); restore prints `<id>  <fingerprint>`.

An archived task keeps its id (ids are never reused) and its links still resolve; it still counts in its epic's progress. It is **read-only**: `task view` shows it (`archived: { at, by }` in `--json`), but `task edit` and `task verify` refuse it (`archived`) with a pointer to `task restore`. Archive refuses an archived task, restore a live one (`not-archived`), and both refuse a target path that exists. Nothing archives in the background — only these commands and the web UI's explicit actions. To delete a task for good, the web UI's Archive screen has Delete permanently; git history keeps it.

## Local web UI

`manatomic tasks serve [--port <n>] [--host <addr>]` serves the data root over HTTP and WebSocket until interrupted — a browser UI plus JSON API for humans reviewing what agents did (default `http://127.0.0.1:4400`; `--port 0` picks a free port). Agents normally don't need it; prefer the CLI.

`manatomic tasks hub [--port <n>]` serves every project in the hub registry from one process, always on 127.0.0.1 (default `http://127.0.0.1:4500`): each project's API under `/p/<id>/api/…` (same answers as `serve`'s `/api/…`), its change events at `/p/<id>/ws`, and `GET /api/hub/projects` (each row carries the project's config.yml `avatar` when set) / `POST /api/hub/projects {path, name?}` to list or register projects, `PATCH /api/hub/projects/:id {name?, path?}` to rename one or point it at another data root, and `DELETE /api/hub/projects/:id` to unregister it (its files stay). The web UI's Settings → Avatar sets the avatar's text and color (`PUT /api/config {avatar}`) and uploads or removes its image (`PUT` / `DELETE /api/avatar`, png/jpeg/gif/webp/svg up to 512 KiB; `GET /api/avatar` serves it). It creates an empty `hub.yml` if there is none, applies `hub add`/`hub remove` without a restart, and lists a missing root as unavailable (`503 project-unavailable` under `/p/<id>/`). A port in use exits 1; a stray word (`hub lst`) exits 2.

`manatomic tasks hub autostart on [--port <n>]` starts the hub now and at every login; `hub autostart off` stops it and removes the entry (nothing installed → exit 0). macOS: LaunchAgent `~/Library/LaunchAgents/com.manatomic.hub.plist`, logs in `~/Library/Logs/manatomic-hub.log`, listed under System Settings → General → Login Items (macOS 13+ shows a "Background Items Added" notice). Linux: `systemd --user` unit `manatomic-hub.service` in `$XDG_CONFIG_HOME/systemd/user/` (else `~/.config/systemd/user/`), logs via `journalctl --user -u manatomic-hub`; a failing `systemctl --user` (no session bus) exits 1 with the unit path and the commands to run by hand. The entry restarts the hub after a crash and runs it with the current `PATH` and `XDG_CONFIG_HOME` — re-run `on` after a `PATH` change. `on` needs the installed binary (exit 1 from a source checkout); `--port 0` exits 2. Windows: not supported yet (exit 1, nothing written).

Both refuse (`403`) a request whose `Host` is not `127.0.0.1`, `localhost` or `[::1]` on their port (`forbidden-host`; `serve --host <addr>` also accepts `<addr>`, and a wildcard `--host 0.0.0.0` or `::` skips only this check), and a write or WebSocket upgrade whose `Origin` is another host (`forbidden-origin`). Requests without an `Origin` — the CLI, curl — pass.

## Hub registry

`hub.yml` is the user-level list of manatomic projects on this machine: `$XDG_CONFIG_HOME/manatomic/hub.yml` (when set and absolute), else `~/.config/manatomic/hub.yml` on macOS and Linux, `%APPDATA%\manatomic\hub.yml` on Windows. It holds a `projects:` list of `{ id, name, root }` (root = absolute data-root path).

| action | command |
|---|---|
| Register a project | `manatomic tasks hub add [path] [--name <name>]` — `path` is a repo folder or a data root (default: the `--root` data root); prints `<id>  <root>`. The id is the repo folder name slugged (`-2`, `-3`, … on collision); re-adding a registered root prints its existing id and exits 0; a path without `config.yml` exits 1 (`not-a-data-root`) |
| Unregister | `manatomic tasks hub remove <id>` — unknown id exits 1 (`not-found`) |
| Rename | `manatomic tasks hub rename <id> <name>` — changes the display name; the id and `/p/<id>/` URLs stay; prints `<id>  <name>` |
| Change the data root | `manatomic tasks hub set-path <id> <path>` — a repo folder or data root, validated like `hub add`; prints `<id>  <root>` |
| List | `manatomic tasks hub list` — `id  name  root` per project, `unavailable` appended when the root has no `config.yml`; `--json`: `[{ id, name, root, available }]` |

A `hub.yml` with bad YAML or a malformed or duplicate id fails with `hub-invalid`, naming the id.

## Field semantics

- **Description** — the *why*: purpose and scope, no implementation detail.
- **Acceptance criteria** — the *what*: outcome-focused checkboxes, numbered 1-based by position in CLI/UI output (never in the file). Each criterion may carry a `verify:` sub-bullet with a shell command; **a criterion without `verify:` is manual**. Continuation lines under a criterion are its **steps** — by convention a manual criterion's numbered walk-through — printed under it by `task view`, exposed as `steps` by `--json`, written by a multi-line `--ac` / `--ac-add`. The checkbox is the single source of truth for "done".
- **Implementation plan** — the *how*; its approval state is the plan-approval marker (below).
- **Decision log** — append-only typed entries (`plan | decision | progress`, each with author and timestamp). This replaces free-form comments: questions asked, recommendations, choices made and progress notes all go here via `--log`. **Record a question and its answer as one `decision` entry in the Q&A convention**, which the web UI renders as question, options, recommendation and choice:

  ```
  Q: Which port should serve use? / Options: 4400, 8080 (free) / Recommendation: 4400 / User chose: 8080 (free)
  ```

  List every option you offered in `Options:`, and write `Recommendation:` and `User chose:` as one option's label, copied word for word, with any reason after ` — ` (`User chose: 8080 (free) — 4400 is taken`); an answer outside the offered options is written as given. That is how the web UI knows which option was chosen and whether the choice went against the recommendation. `Options:` split on `;` outside parentheses when there is one, else on commas outside parentheses; `Chosen:` may stand for `User chose:`, and a note may follow the choice. An entry that records `User chose:` or `Chosen:` counts as **answered** — an unanswered question from an agent is what puts a task in the web UI's Needs you. Keep one question per entry: an entry with several reads as plain text.
- **Summary** — short PR-ready completion summary.

## Verify evidence

`manatomic tasks task verify MAN-1 --by @claude` runs each criterion's `verify:` command (`--ac <n>` limits to one) and records the outcome as an `evidence:` sub-bullet — `pass|fail`, who, when, the commit sha, and one line of output. Failing commands are evidence, not errors: the criterion reads `failing`, its checkbox is untouched. A pass checks the box only when `verify.auto_check` is set; otherwise check it separately with `--check-ac <n>`. Manual criteria are never run.

- Verify is **opt-in**: without `verify.enabled: true` in `config.yml` the CLI refuses with `verify-disabled`. `config show` reveals the setting (plus `timeout_seconds` and `max_output`).
- Commands run with `sh -c` in the **parent directory of the data root** (`<repo>/manatomic` → `<repo>`), so write them repo-relative. When the data root lives outside the repo (e.g. a shared `--root ~/Documents/tasks`), pass `--cwd <repo root>` — or `--cwd .` from the clone's root — so commands and the commit sha come from that repo. A relative `--cwd` resolves against the current directory; a missing path or a file exits 2.
- Prefer recorded evidence over trusting checkboxes: `task view --json` reports each criterion's derived `state` — `manual`, `pending` (no evidence yet), `verified` or `failing`. Plain `task view` prints the checkbox, verify command and steps; evidence sub-bullets live in the file.
- **Manual evidence**: after a person walked a manual criterion's steps, `task verify MAN-1 --ac 3 --manual --by @najdan` ticks it and records `pass by @najdan at <ts>: manual check` — who checked, and when. It runs nothing, so it works with verify disabled, and refuses a criterion that has a command (`ac-has-verify`). The web UI's **Mark verified** writes the same line. The state stays `manual`; a plain `--check-ac` still ticks without evidence. Only record it for a check the person actually made.

## Plan approval

Frontmatter keys `plan_approved_by` + `plan_approved_at` (both present ⇒ plan approved) record who approved the implementation plan and when. **Replacing the plan body clears the marker automatically**, so approval always refers to the plan as it was read. To record an approval the user gave:

```sh
manatomic tasks task edit MAN-1 --set plan_approved_by=@najdan --set "plan_approved_at=2026-09-20T15:00:00Z"
manatomic tasks task edit MAN-1 --log decision --author @najdan --message "Plan approved."
```

`task view --json` reports it as `planApproval`. Check it before implementing: a task with a plan but no marker is waiting on a human (unless `features.plan_approval` is `false`).

## Seen marker

Frontmatter keys `seen_by` + `seen_at` record that a person has seen a task's latest log entry — the web UI's Agents → Needs you writes them with **Mark as seen**, and the task stays out of Needs you until a log entry newer than `seen_at` arrives. Writing them does not touch `updated`. `task view --json` reports it as `seen: { by, at } | null`. Agents leave them alone; to bring a task back to someone's attention, append a new log entry.

## Fingerprints and write conflicts

Every read returns a `fingerprint` of the file's exact bytes (`task view` prints it as a field) and every mutation prints the new one (`<id>  <fingerprint>`). Writes are **last write wins** — the CLI never blocks a write — but the task-mutating commands (`task edit`, `task verify`) take `--if-match <fingerprint>`: pass the fingerprint from your last `task view` or mutation, and when the file changed under you the CLI still writes and tells you — a `conflict:` warning on stderr in plain mode, a `"conflict": true` field in the `--json` payload (stderr stays reserved for errors there). Without `--if-match` there is no check and no warning. When other writers may be active (the web UI, an editor, another agent), always pass it, and re-read before editing further when it flags a conflict; the web UI gets the same signal over HTTP (`If-Match` header). Hand-editing task files is forbidden for the same reason: it races the CLI/UI writers and skips validation.

## Known gaps and workarounds

- **Repeated `--remove-ac` numbers all refer to the pre-removal numbering** (`--remove-ac 1 --remove-ac 3` removes the original #1 and #3), removal runs after `--check-ac`/`--uncheck-ac` in the same call, and one out-of-range number fails the call before anything is removed.
- **No `doc update`** — deliberate: edit doc bodies as files (see the rule split above).

## If `manatomic` is not installed

```sh
curl -fsSL https://github.com/manatomic/md-tasks-releases/releases/latest/download/install.sh | bash
```

Installs a single-file binary as `~/.local/bin/manatomic` (`MANATOMIC_VERSION=v1.2.3` pins a tag, `MANATOMIC_INSTALL_DIR` another directory). The file format spec ships in the same repo under `docs/manatomic-file-format.md`.
