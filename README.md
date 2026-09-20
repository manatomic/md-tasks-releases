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
manatomic tasks task view MAN-1                # fields, acceptance criteria, decision log
manatomic tasks task edit MAN-1                # change a task
manatomic tasks task verify MAN-1              # run AC verify commands, record evidence
```

Docs and decision records follow the same pattern: `manatomic tasks doc
create|list|view` and `manatomic tasks decision create|list|view`.

Browse and edit everything in a local web UI (served until interrupted):

```sh
manatomic tasks serve
```

Every command takes `--root <dir>` to point at a different data root and
`--json` for JSON output (plain text is the default). `manatomic tasks --help`
lists all commands; each command has its own `--help`.

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
