# cctools — Bundle Design

**Date:** 2026-06-24
**Status:** Approved design (pre-implementation)
**Author:** paulius

## 1. Purpose

Bundle three independent Claude Code workflow helpers into a single repository,
`plabanauskis/cctools`, with one self-contained installer and a small management
command. The goal is a clean, public-ready (eventually) toolset for people working
with Claude Code in the Linux terminal. Not commercial — just useful and well-kept.

The three tools:

| Tool | What it does | Weight | Deps | Platform |
|------|--------------|--------|------|----------|
| **cchat** | Opens Claude Code in a fresh ephemeral `/tmp` dir for throwaway chats | Featherweight (~5 lines) | `claude` | linux, macos |
| **ccsession** | `fzf` picker to list and resume any Claude Code session without `cd`-ing | Light (~9KB bash) | `fzf`, `jq`, `claude`, GNU coreutils | linux, macos |
| **ccbox** | Sandboxed autonomous Claude Code via Docker + sysbox (path-identical host mirror) | Heavy (bash + Dockerfile, 5GB image) | `docker`, `sysbox-ce`, `claude` | linux (amd64) |

They cohere as "Claude Code workflow helpers," but differ sharply in weight and
platform support. The design's central tension — and the reason for opt-in,
dep-aware install — is that ccbox is heavy and Linux-only while cchat/ccsession
are tiny and portable. A user who only wants `ccsession` must never be forced
into ccbox's Docker world.

## 2. Repo layout

```
cctools/
├── README.md                 # front door: pitch, install one-liner, tool matrix, uninstall promise
├── LICENSE                   # MIT
├── install.sh                # the curl|bash bootstrap (also the dev/manual entrypoint)
├── bin/
│   └── cctools               # management command (lifecycle only — NOT a tool dispatcher)
├── tools/
│   ├── cchat/
│   │   ├── cchat             # standalone script (converted from the ~/.zshrc function)
│   │   ├── tool.manifest
│   │   ├── README.md
│   │   ├── VERSION           # 1.0.0
│   │   ├── CHANGELOG.md
│   │   └── tests/
│   ├── ccsession/
│   │   ├── ccsession
│   │   ├── tool.manifest
│   │   ├── README.md
│   │   ├── VERSION           # 1.0.0
│   │   ├── CHANGELOG.md
│   │   └── tests/
│   └── ccbox/
│       ├── bin/ccbox
│       ├── Dockerfile
│       ├── entrypoint.sh
│       ├── tool.manifest
│       ├── README.md
│       ├── VERSION           # 2.0.0
│       ├── CHANGELOG.md
│       ├── docs/             # existing design/threat-model spec
│       └── test/             # smoke.sh
├── scripts/
│   ├── check.sh              # shellcheck + shfmt + all tool tests (the local "CI")
│   ├── release.sh            # optional helper: bump VERSION+CHANGELOG, tag, gh release
│   └── dev-setup.sh          # one-time: git config core.hooksPath .githooks
├── .githooks/
│   └── pre-push              # runs scripts/check.sh; blocks push on failure
└── docs/superpowers/
    ├── specs/                # this document
    └── plans/                # implementation plan (written next)
```

Each tool is **self-contained** under `tools/<name>/`. Adding a future 4th tool is
just dropping a new `tools/<name>/` directory with a `tool.manifest` — the installer
and `cctools` command stay generic and need no edits.

**Explicitly dropped from ccbox during import:** `packaging/build-deb.sh`, `dist/*`
(the `.deb` channel is gone — see §4), and `ccbox-implementation-plan.md` (stale).

## 3. The tool manifest (the glue)

A small, sourceable `tools/<name>/tool.manifest` per tool keeps `install.sh` and
`bin/cctools` generic — they read manifests rather than hard-coding each tool.

```sh
# tools/ccsession/tool.manifest
NAME=ccsession
ENTRYPOINT=ccsession                 # path (relative to tool dir) symlinked onto PATH
COMMANDS="ccsession"                 # command name(s) the symlink creates
DEPS="fzf jq claude"                 # binaries checked at install + by `cctools doctor`
PLATFORM="linux macos"               # which OSes this tool supports
DESC="fzf picker to resume any Claude Code session"
POST_ENABLE=""                       # message printed after enabling (e.g. ccbox build hint)
```

Examples of the variations:
- **cchat**: `ENTRYPOINT=cchat`, `DEPS="claude"`, `PLATFORM="linux macos"`.
- **ccbox**: `ENTRYPOINT=bin/ccbox`, `DEPS="docker claude"`, `PLATFORM="linux"`,
  `POST_ENABLE="Run 'ccbox build' once to build the sandbox image (~5 min, ~5GB)."`
  (sysbox is checked by `ccbox doctor`, which already exists in the tool.)

Manifest is the single contract between a tool and the bundle's plumbing.

## 4. Install model

Primary (and only published) channel — a thin `curl|bash` bootstrap over a
self-contained prefix:

```bash
curl -fsSL https://raw.githubusercontent.com/plabanauskis/cctools/main/install.sh | bash
```

`install.sh` behavior, with **no `sudo`**, touching only two user-owned locations:

- `${CCTOOLS_HOME:-$HOME/.local/share/cctools}` — a git clone of the repo (single source of truth)
- `$HOME/.local/bin` — symlinks, one per enabled command

Steps:
1. Detect platform/arch.
2. Clone the repo into `$CCTOOLS_HOME`, or `git pull` if already present (idempotent).
3. **Select tools:**
   - `--all` or `--tools=ccsession,cchat` (non-interactive override), else
   - interactive prompt read from `/dev/tty` (works even under `curl|bash`), else
   - if no TTY: install every tool whose `PLATFORM` matches the host **and** whose
     `DEPS` are all present; **print exactly which tools were skipped and the command
     to enable each later** (no silent skipping).
4. Per selected tool: check `DEPS` (missing → warn; still linkable with `--force`),
   symlink `ENTRYPOINT` → `~/.local/bin/<command>`, print `POST_ENABLE` if set.
5. Always symlink `bin/cctools` → `~/.local/bin/cctools`.
6. If `~/.local/bin` is not on `PATH`, warn and print the exact line to add.
7. Print a summary: enabled tools, skipped tools + why, next steps.

Properties this guarantees (directly answering the "don't litter the host" concern):
- **No system pollution / no sudo** — nothing in `/usr`, `/etc`; no daemons. Same
  pattern as `rustup` (`~/.cargo`), `fnm`, `nvm`, Claude Code's native installer.
- **Auditable** — the one-liner only fetches `install.sh`; README documents the
  read-first path (`curl -O … && less install.sh && bash install.sh`).
- **Updatable** — re-run, or `cctools update` = `git -C $CCTOOLS_HOME pull` + re-link.
- **Cleanly reversible** — `cctools uninstall` removes exactly the symlinks + the
  prefix dir. Because nothing else was created, the host is pristine afterward.

**Dev / manual install** (clone + run, no curl): `git clone … && ./install.sh` — the
same script, run locally.

No `.deb`, no Homebrew tap. (Homebrew could be added later as a thin tap over the
same repo if there is ever demand — out of scope now.)

## 5. CLI surface

The three tools are invoked by **their own names** — `cchat`, `ccbox`, `ccsession`.
There is **no `cctools <subcommand>` dispatcher** for running tools.

`cctools` is a **management command only**:

| Command | Action |
|---------|--------|
| `cctools list` | Tools, enabled/disabled state, versions, dep status |
| `cctools doctor [tool]` | Check `DEPS` + `PLATFORM` for all tools or one |
| `cctools enable <tool>` | Symlink the tool's command(s) into `~/.local/bin` |
| `cctools disable <tool>` | Remove those symlinks |
| `cctools update` | `git pull` in `$CCTOOLS_HOME`, then re-link enabled tools |
| `cctools uninstall` | Remove all symlinks + the prefix dir (prompts first) |
| `cctools version [tool]` | Print version(s) from `VERSION` files |

`cctools uninstall` defers ccbox image/volume cleanup to `ccbox uninstall` (it tells
the user to run that first if the image/volumes exist; it never deletes Docker data).

### cchat as a standalone script

cchat moves from a `~/.zshrc` function to a real script on `PATH`. The function form
existed only so the `cd` happened in a subshell, leaving the parent shell's CWD
unchanged. A standalone script already runs in its own process, so the parent shell's
CWD is untouched automatically — the script is simpler than the function:

```sh
#!/usr/bin/env bash
set -euo pipefail
dir="$(mktemp -d "${TMPDIR:-/tmp}/cchat.XXXXXX")"
cd "$dir"
exec claude "$@"
```

`--help` is supported. The temp dir persists until reboot (same as today).

## 6. Versioning & releases (per-tool, local)

- Each tool has `tools/<name>/VERSION` (plain semver) + `CHANGELOG.md`.
- Seeded to current reality: **ccbox 2.0.0, ccsession 1.0.0, cchat 1.0.0**.
- Git tags are namespaced per tool: `<tool>-vX.Y.Z` (e.g. `ccbox-v2.1.0`).
- A release is a **local/manual** action (no CI): bump that tool's `VERSION` +
  `CHANGELOG.md`, tag `<tool>-vX.Y.Z`, and (optionally) `gh release create`.
- `scripts/release.sh <tool> <version>` is an optional helper that automates the
  bump → tag → `gh release` sequence for one tool.
- `cctools version` reads the `VERSION` files directly.

## 7. Local checks (replaces CI)

No GitHub Actions. All verification runs locally and must pass **before a push**:

- `scripts/check.sh` runs:
  - **shellcheck** over `install.sh`, `bin/cctools`, `scripts/*.sh`, and every tool's scripts;
  - **`shfmt --diff`** as an enforced formatting gate (push is blocked on any diff);
  - each tool's test suite (ccsession's bash suite; cchat's tests; ccbox's
    non-container checks).
- `.githooks/pre-push` runs `scripts/check.sh` and blocks the push on failure.
- `scripts/dev-setup.sh` wires it up once: `git config core.hooksPath .githooks`.

**Honest caveat (carried from ccbox):** ccbox's `test/smoke.sh` needs a running
Docker + sysbox runtime, which a generic check environment may lack. `check.sh` runs
shellcheck + any non-container checks for ccbox always, and runs the full smoke test
only when Docker+sysbox are detected (otherwise it prints `SKIP: ccbox smoke (no
sysbox)` — never a silent pass). The full smoke remains a documented manual gate.

## 8. Documentation

- **Root `README.md`** — the front door:
  - one-paragraph pitch of cctools;
  - the install one-liner + the read-first variant;
  - a **tool matrix** (tool · what it does · deps · platform) like §1;
  - "What gets installed / how to uninstall" stated up top (the anti-litter promise);
  - per-tool quick links; a security note linking ccbox's threat model.
- **Per-tool `README.md`** — the existing ones, relocated under `tools/<name>/` and
  trimmed so install instructions point to the root installer + `cctools enable <tool>`
  rather than duplicating per-tool install steps.
- No `CONTRIBUTING.md` and no `DEVELOPMENT.md` — no external contributions expected;
  the release and check procedures are self-evident from `scripts/`.

## 9. Migration & publish

- **Create `plabanauskis/cctools` as PRIVATE**: `gh repo create plabanauskis/cctools --private`.
  Flip to public manually when ready.
- **Import** the working trees of ccbox + ccsession + cchat (extracted from
  `~/.zshrc`) under `tools/`, as a **single import commit** (fresh start — no history
  preservation from the old repos).
- **Legacy repos (`plabanauskis/ccbox`, `plabanauskis/ccsession`): no action.** The
  maintainer deletes them manually afterward.
- **Local environment:** remove the `cchat` function from `~/.zshrc` (a shell function
  shadows a PATH command, so it must go for the new `cchat` script to win); ensure
  `~/.local/bin` is on `PATH`.

## 10. Implementation phases

The implementation plan (written next, in `docs/superpowers/plans/`) will sequence:

0. **Scaffold** the repo: layout, `LICENSE` (MIT), root `README.md` skeleton,
   `.githooks/`, `scripts/` stubs.
1. **Import** the three tools under `tools/`: copy ccbox (minus packaging/dist/stale
   plan) and ccsession; convert cchat to a standalone script; seed each tool's
   `VERSION` + `CHANGELOG.md`; write each `tool.manifest`.
2. **Plumbing:** build `install.sh` + `bin/cctools` (generic, manifest-driven).
3. **Local checks:** `scripts/check.sh`, `.githooks/pre-push`, `scripts/dev-setup.sh`,
   and the optional `scripts/release.sh`.
4. **Docs:** root README front door, per-tool README trims.
5. **Publish:** create the private GitHub repo, single import commit, push; remove the
   `cchat` function from `~/.zshrc`; (later, manually) cut per-tool releases and delete
   the legacy repos.

## Out of scope (v1)

- GitHub Actions / hosted CI.
- `.deb`, Homebrew, or any OS package manager channel.
- A hosted docs website.
- A `cctools <subcommand>` dispatcher for running tools.
- Preserving the legacy repos' git history.
- macOS support for ccbox (remains Linux/amd64).
