# md-tasks

Releases of **md-tasks** by manatomic — a markdown task system where plain
files in your repo are the single source of truth. Tasks, docs and decision
records live as markdown in a data root you commit alongside your code, and
the `manatomic` CLI works with them from the terminal or a local web UI.

## Install

Each release ships single-file binaries with the Bun runtime included — nothing
else to install:

```sh
curl -fsSL https://github.com/manatomic/md-tasks-releases/releases/latest/download/install.sh | bash
```

The script detects your OS and architecture, downloads the matching binary from
the latest release and installs it as `~/.local/bin/manatomic`. Environment
variables:

| Variable | Effect |
| --- | --- |
| `MANATOMIC_VERSION=v1.2.3` | install that tag instead of the latest release |
| `MANATOMIC_INSTALL_DIR=...` | install somewhere other than `~/.local/bin` |

**Windows:** download `manatomic-windows-x64.exe` from the
[releases page](https://github.com/manatomic/md-tasks-releases/releases) directly.

Check the installed version with `manatomic --version`.

## Usage

Scaffold a data root in your project (default directory `./manatomic`; the
prefix is used for task ids like `MAN-1`):

```sh
manatomic tasks init --prefix MAN
```

This creates `config.yml` plus `tasks/`, `docs/` and `decisions/` — all plain
markdown, meant to be committed with your code.

Work with tasks:

```sh
manatomic tasks task create "Add login page"   # create a task, prints its id
manatomic tasks task list                      # one line per task, filters AND-combine
manatomic tasks task view MAN-1                # fields, subtasks, acceptance criteria, decision log
manatomic tasks task edit MAN-1                # change a task
manatomic tasks task verify MAN-1              # run AC verify commands, record evidence
```

Acceptance criteria travel with the task: `task create` takes repeatable
`--ac "criterion"` / `--ac-verify "cmd"` pairs (each `--ac-verify` attaches to
the nearest preceding `--ac`), and `task edit` adds (`--ac-add`, the same
repeatable pairing with `--ac-verify`), checks (`--check-ac`), unchecks
(`--uncheck-ac`) and removes (`--remove-ac`, the rest renumber) them later. A task's Summary — its short PR-ready completion note —
is replaced with `task edit --summary` or extended with repeatable
`--append-summary` flags (one paragraph per use), so progress notes can be
added without resending the existing text. Each command's `--help` lists
every flag.

Docs and decision records follow the same pattern: `manatomic tasks doc
create|list|view` and `manatomic tasks decision create|list|view`. Search
spans all three — a case-insensitive substring match over ids, titles and
body text:

```sh
manatomic tasks search "login"
```

Browse and edit everything in a local web UI (served until interrupted, at
`http://127.0.0.1:4400`):

```sh
manatomic tasks serve
```

Keep a list of the manatomic projects on your machine — the hub registry, a
`hub.yml` file in your user config folder: `$XDG_CONFIG_HOME/manatomic/` when
that is set (an absolute path), else `~/.config/manatomic/` on macOS and Linux,
and `%APPDATA%\manatomic\` on Windows:

```sh
manatomic tasks hub add                     # register ./manatomic (or --root <dir>)
manatomic tasks hub add ../api --name API   # a repo folder or a data root
manatomic tasks hub list                    # id, name and root per project
manatomic tasks hub remove api              # unregister by id
```

A project's id is its repo folder name as a slug (`-2`, `-3`, … when taken),
and its name is the folder name unless `--name` is given. `hub list` marks a
project `unavailable` when its folder no longer has a `config.yml`. Once
`hub.yml` exists, `manatomic tasks init` adds each new data root to it
automatically; `init` never creates the file itself.

Serve every registered project from one process, at one fixed address —
`http://127.0.0.1:4500`, each project under `/p/<id>/`:

```sh
manatomic tasks hub                         # until interrupted; --port <n> moves it
```

The hub always binds `127.0.0.1`, creates an empty `hub.yml` if there is none,
and picks up `hub add` and `hub remove` while it runs; a project whose folder
is missing is shown as unavailable without affecting the others.

Both `serve` and `hub` only answer requests addressed to this machine
(`127.0.0.1`, `localhost` or `[::1]` on their port) and refuse writes and
WebSocket connections from other websites (an `Origin` check), so a page you
visit cannot reach your tasks. `serve --host 0.0.0.0` (or `::`) skips the
address check and exposes the data root to your network, with no auth.

Every command takes `--root <dir>` to point at a different data root and
`--json` for JSON output (plain text is the default). `manatomic tasks --help`
lists all commands; each command has its own `--help`.

## Agent skills (Claude Code plugin)

This repo is also a [Claude Code](https://claude.com/claude-code) plugin
marketplace. The `md-tasks` plugin ships agent skills for the CLI, versioned
with the release that published them:

```sh
claude plugin marketplace add manatomic/md-tasks-releases
claude plugin install md-tasks@manatomic
```

(Or run `/plugin install md-tasks@manatomic` from inside a Claude Code
session after adding the marketplace.)

Skills:

- `md-tasks:cli` — full CLI reference: every command, field semantics, verify
  evidence, plan approval, fingerprints/conflicts, and known gaps with
  workarounds. Also auto-triggers when an agent works in a repo with a
  manatomic data root.
- `md-tasks:migrate-from-backlog` — one-shot, verified migration of a
  [backlog.md](https://backlog.md) data root (`backlog/`) into a manatomic
  data root: ids, epics, acceptance criteria, comments-to-decision-log and
  docs, with `backlog/` removed only after the result checks out.

The `docs/` folder carries the published documentation:
[manatomic file format](docs/manatomic-file-format.md), the spec for the
markdown/YAML files under a data root.

## Release assets

| Asset | Description |
| --- | --- |
| `manatomic-darwin-arm64` | macOS, Apple Silicon |
| `manatomic-darwin-x64` | macOS, Intel |
| `manatomic-linux-arm64` | Linux, ARM64 |
| `manatomic-linux-x64` | Linux, x86-64 |
| `manatomic-windows-x64.exe` | Windows, x86-64 |
| `SHA256SUMS.txt` | checksums for the binaries above |
| `install.sh` | the install script, pinned to that release |

To verify a download against the checksums:

```sh
curl -fsSLO https://github.com/manatomic/md-tasks-releases/releases/latest/download/SHA256SUMS.txt
sha256sum --check --ignore-missing SHA256SUMS.txt
```

## Issues

Bug reports and feature requests are welcome in
[Issues](https://github.com/manatomic/md-tasks-releases/issues).
