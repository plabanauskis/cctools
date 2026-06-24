# cctools Bundle Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Bundle three Claude Code workflow helpers (`cchat`, `ccsession`, `ccbox`) into the `plabanauskis/cctools` repo with one self-contained, manifest-driven installer and a `cctools` management command.

**Architecture:** Each tool is self-contained under `tools/<name>/` with a sourceable `tool.manifest`. Generic plumbing — `install.sh` (curl|bash bootstrap) and `bin/cctools` (lifecycle command) — reads manifests instead of hard-coding tools, so a 4th tool is just a new directory. Install touches only two user-owned paths (`$CCTOOLS_HOME`, `~/.local/bin`); no sudo, fully reversible. Verification is local (`scripts/check.sh` + a `pre-push` hook); no CI.

**Tech Stack:** Bash (5.x), POSIX coreutils, `git`. Tool deps: `claude` (all), `fzf`+`jq` (ccsession), `docker`+`sysbox-ce` (ccbox). Lint/format gate: `shellcheck` + `shfmt` (run-if-present). ccbox image: Debian + Docker (`Dockerfile`).

## Global Constraints

Every task's requirements implicitly include these (copied from the spec):

- **No sudo. Two user-owned locations only:** `${CCTOOLS_HOME:-$HOME/.local/share/cctools}` (git clone = single source of truth) and `$HOME/.local/bin` (symlinks, one per enabled command). Nothing in `/usr`, `/etc`; no daemons.
- **Manifest-driven:** `install.sh` and `bin/cctools` must read `tools/<name>/tool.manifest` — never hard-code a tool list. Adding a tool = dropping a new `tools/<name>/` dir with a manifest, no plumbing edits.
- **Manifest fields:** `NAME`, `ENTRYPOINT` (path relative to tool dir, symlinked onto PATH), `COMMANDS` (space-separated command names the symlink creates), `DEPS` (space-separated binaries), `PLATFORM` (space-separated of `linux macos`), `DESC`, `POST_ENABLE` (message after enabling).
- **Per-tool versioning:** `tools/<name>/VERSION` (plain semver) + `CHANGELOG.md`. Seeded versions: **ccbox 2.0.1** (reconciled with `CCBOX_VERSION` already in `bin/ccbox`; the spec's "2.0.0" predates the 2.0.1 patch — VERSION must equal what `ccbox version` prints), **ccsession 1.0.0**, **cchat 1.0.0**. Git tags namespaced `<tool>-vX.Y.Z`.
- **CLI surface:** tools run by their own names (`cchat`, `ccsession`, `ccbox`). `cctools` is management-only: `list`, `doctor [tool]`, `enable <tool>`, `disable <tool>`, `update`, `uninstall`, `version [tool]`. No `cctools <tool>` run-dispatcher.
- **Out of scope (do NOT build):** GitHub Actions/CI, `.deb`, Homebrew, hosted docs, a run-dispatcher, legacy git-history preservation, macOS support for ccbox.
- **Strict-mode + sourceable pattern:** plumbing scripts must be unit-testable by sourcing. Use a guard so strict mode + `main` run only when executed, not when sourced (details in each task). Target bash ≥ 4.4 (machine has 5.2): `"${arr[@]}"` on an empty array under `set -u` is safe.
- **Imports drop:** ccbox `packaging/`, `dist/`, `ccbox-implementation-plan.md` (per spec), plus ccbox `CLAUDE.md`, `LICENSE`, `.gitignore`, `.claude/` (stale/redundant — release process is now repo-wide; root LICENSE covers attribution). ccsession `HANDOFF.md` (confirmed by maintainer: already implemented, irrelevant), `docs/`, `.gitignore` dropped. ccbox `assets/` and `docs/` ARE kept (README references them; root README links ccbox's threat-model spec).
- **Environment available now:** bash 5.2, `fzf`/`jq`/`docker`/`git`/`gh`/`claude`/`node` present, sysbox present, **`shellcheck` + `shfmt` 3.8.0 installed** (so `check.sh` ENFORCES both gates: every shipped script must pass `shellcheck -x` and be `shfmt -i 2 -ci` clean — format each script with `shfmt -i 2 -ci -w` before committing). The ccbox image is not built, so `check.sh`'s ccbox smoke SKIPs (no image) — never a silent pass.
- **Shared plumbing lib:** `lib/cctools-common.sh` holds the manifest-reading helpers (`current_os`, `list_tools`, `load_manifest`, `platform_ok`, `missing_deps`, `tool_version`, `entrypoint_path`), parameterized by `CCTOOLS_TOOLS_DIR`. Both `bin/cctools` (sources at top-level) and `install.sh` (sources after cloning) use it — no duplication. (Approved deviation from the §2 layout: a `lib/` for shared plumbing serves the spec's "generic, manifest-driven, thin bootstrap" intent better than copy-paste.)

---

## File Structure

**New plumbing & docs**
- `LICENSE` — MIT + claudebox attribution line.
- `.gitignore` — editor/OS cruft + test scratch.
- `README.md` — front door (pitch, install one-liner, tool matrix, uninstall promise, links).
- `lib/cctools-common.sh` — shared manifest-reading helpers (sourced by both `bin/cctools` and `install.sh`); see Global Constraints.
- `install.sh` — curl|bash bootstrap; clones prefix, sources the lib post-clone, selects + links tools. Sourceable for tests via `CCTOOLS_TEST_SOURCE=1`.
- `bin/cctools` — management command; sources the lib at top-level. Sourceable for tests (BASH_SOURCE guard).
- `scripts/check.sh` — local CI: shellcheck + shfmt (enforced; both installed) + all test suites + gated ccbox smoke.
- `scripts/dev-setup.sh` — `git config core.hooksPath .githooks`.
- `scripts/release.sh` — per-tool VERSION+CHANGELOG bump, tag, optional `gh release`. Bump fns sourceable.
- `.githooks/pre-push` — runs `scripts/check.sh`.
- `tests/cctools.test.sh`, `tests/install.test.sh`, `tests/release.test.sh` — plumbing test suites (root `tests/` extends the spec layout; required for TDD of the plumbing).

**Per-tool (each self-contained under `tools/<name>/`)**
- `tools/cchat/`: `cchat` (new standalone script), `tool.manifest`, `VERSION`, `CHANGELOG.md`, `README.md`, `tests/cchat.test.sh`.
- `tools/ccsession/`: `ccsession` (copied), `tests/ccsession.test.sh` (copied), `tool.manifest`, `VERSION`, `CHANGELOG.md`, `README.md` (trimmed).
- `tools/ccbox/`: `bin/ccbox`, `Dockerfile`, `entrypoint.sh` (copied), `test/smoke.sh` (copied), `assets/` (copied), `docs/` (copied), `tool.manifest`, `VERSION`, `CHANGELOG.md`, `README.md` (trimmed).

**Manifest contract (shared shape, one per tool)** — sourced as shell; later tasks rely on these exact variable names: `NAME ENTRYPOINT COMMANDS DEPS PLATFORM DESC POST_ENABLE`.

---

## Task 1: Repo foundation — LICENSE + .gitignore

**Files:**
- Create: `LICENSE`
- Create: `.gitignore`

**Interfaces:**
- Produces: nothing consumed programmatically; establishes the repo root files.

- [ ] **Step 1: Create `LICENSE`** (MIT, with the claudebox attribution carried from ccbox)

```
MIT License

Copyright (c) 2026 Paulius Labanauskis

Permission is hereby granted, free of charge, to any person obtaining a copy
of this software and associated documentation files (the "Software"), to deal
in the Software without restriction, including without limitation the rights
to use, copy, modify, merge, publish, distribute, sublicense, and/or sell
copies of the Software, and to permit persons to whom the Software is
furnished to do so, subject to the following conditions:

The above copyright notice and this permission notice shall be included in all
copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE
AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM,
OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE
SOFTWARE.

Portions of ccbox adapted from RchGrav/claudebox (MIT).
```

- [ ] **Step 2: Create `.gitignore`**

```
# Editor / OS cruft
*.swp
*~
.DS_Store

# Test scratch
**/tests/tmp/
**/test/tmp/
```

- [ ] **Step 3: Commit**

```bash
git add LICENSE .gitignore
git commit -m "chore: add LICENSE (MIT) and .gitignore"
```

---

## Task 2: Import cchat as a standalone tool

cchat converts from a `~/.zshrc` function to a real script (a standalone script already runs in its own process, so the parent shell's CWD is untouched).

**Files:**
- Create: `tools/cchat/cchat`
- Create: `tools/cchat/tool.manifest`
- Create: `tools/cchat/VERSION`
- Create: `tools/cchat/CHANGELOG.md`
- Create: `tools/cchat/README.md`
- Test: `tools/cchat/tests/cchat.test.sh`

**Interfaces:**
- Produces: command `cchat`; entrypoint `cchat`; manifest vars (`DEPS="claude"`, `PLATFORM="linux macos"`).

- [ ] **Step 1: Write the failing test** — `tools/cchat/tests/cchat.test.sh`

```bash
#!/usr/bin/env bash
# Tests for cchat: it must make a fresh temp dir, cd into it, and exec `claude`
# there, passing through args. We stub `claude` with a recorder on PATH so the
# real CLI never launches.
set -u
HERE="$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)"
CCHAT="$HERE/../cchat"

PASS=0
FAIL=0
assert_contains() { # <haystack> <needle> <msg>
  if [[ "$1" == *"$2"* ]]; then PASS=$((PASS + 1))
  else FAIL=$((FAIL + 1)); printf 'FAIL: %s\n  [%s] does not contain [%s]\n' "$3" "$1" "$2"; fi
}
assert_eq() { # <got> <want> <msg>
  if [ "$1" = "$2" ]; then PASS=$((PASS + 1))
  else FAIL=$((FAIL + 1)); printf 'FAIL: %s\n  got:  [%s]\n  want: [%s]\n' "$3" "$1" "$2"; fi
}

# Sandbox: fake claude + a private TMPDIR so we never touch the real /tmp/cchat.*
SANDBOX="$(mktemp -d)"
trap 'rm -rf "$SANDBOX"' EXIT
FAKEBIN="$SANDBOX/bin"
mkdir -p "$FAKEBIN"
cat >"$FAKEBIN/claude" <<'SH'
#!/usr/bin/env bash
echo "CWD=$PWD"
echo "ARGS=$*"
SH
chmod +x "$FAKEBIN/claude"

# Run cchat with the stub claude first on PATH and an isolated TMPDIR.
out="$(TMPDIR="$SANDBOX" PATH="$FAKEBIN:$PATH" bash "$CCHAT" --model opus -p hi)"
assert_contains "$out" "CWD=$SANDBOX/cchat." "cchat: claude runs inside a fresh cchat.* dir under TMPDIR"
assert_contains "$out" "ARGS=--model opus -p hi" "cchat: passes args straight through to claude"

# --help prints usage and does NOT launch claude.
help_out="$(PATH="$FAKEBIN:$PATH" bash "$CCHAT" --help)"
help_rc=$?
assert_eq "$help_rc" "0" "cchat: --help exits 0"
assert_contains "$help_out" "cchat" "cchat: --help mentions the tool name"
case "$help_out" in *CWD=*) FAIL=$((FAIL + 1)); echo "FAIL: --help must not launch claude";; *) PASS=$((PASS + 1));; esac

printf '\n%d passed, %d failed\n' "$PASS" "$FAIL"
[ "$FAIL" -eq 0 ]
```

- [ ] **Step 2: Run the test to verify it fails**

Run: `bash tools/cchat/tests/cchat.test.sh`
Expected: FAIL (the `cchat` script does not exist yet — `bash: .../cchat: No such file`).

- [ ] **Step 3: Create `tools/cchat/cchat`**

```bash
#!/usr/bin/env bash
# cchat — open Claude Code in a fresh, ephemeral temp dir for a throwaway chat.
# A standalone script runs in its own process, so the parent shell's CWD is
# untouched (no subshell trick needed, unlike the old ~/.zshrc function).
set -euo pipefail

case "${1:-}" in
  -h | --help)
    cat <<'EOF'
cchat — open Claude Code in a fresh ephemeral temp dir for a throwaway chat.

Usage:
  cchat [claude args...]   Make a new temp dir, cd into it, and exec 'claude'.
  cchat --help             Show this help.

The temp dir is created under $TMPDIR (default /tmp) and persists until reboot,
the same as before. All arguments are passed straight through to 'claude'.
EOF
    exit 0
    ;;
esac

dir="$(mktemp -d "${TMPDIR:-/tmp}/cchat.XXXXXX")"
cd "$dir"
exec claude "$@"
```

- [ ] **Step 4: Make it executable and run the test to verify it passes**

Run: `chmod +x tools/cchat/cchat && bash tools/cchat/tests/cchat.test.sh`
Expected: PASS — `4 passed, 0 failed` (CWD-under-TMPDIR, args, --help exit 0, --help no-launch).

- [ ] **Step 5: Create `tools/cchat/tool.manifest`**

```sh
# cchat — open Claude Code in a fresh ephemeral /tmp dir for throwaway chats.
NAME=cchat
ENTRYPOINT=cchat
COMMANDS="cchat"
DEPS="claude"
PLATFORM="linux macos"
DESC="Open Claude Code in a fresh ephemeral /tmp dir for throwaway chats"
POST_ENABLE=""
```

- [ ] **Step 6: Create `tools/cchat/VERSION`**

```
1.0.0
```

- [ ] **Step 7: Create `tools/cchat/CHANGELOG.md`**

```markdown
# Changelog — cchat

All notable changes to cchat are documented here. Versions follow [semver](https://semver.org).

## 1.0.0 — 2026-06-24

- Initial release as a standalone script in the cctools bundle (converted from a
  `~/.zshrc` shell function). Opens Claude Code in a fresh ephemeral temp dir;
  `--help` supported; args pass straight through to `claude`.
```

- [ ] **Step 8: Create `tools/cchat/README.md`**

```markdown
# cchat

Open Claude Code in a fresh, throwaway directory — for quick questions you don't
want cluttering a real project, and that you don't need to keep.

`cchat` makes a new temp dir under `$TMPDIR` (default `/tmp`), `cd`s into it, and
`exec`s `claude` there. Because it runs as its own process, your current shell's
working directory is untouched. The temp dir persists until reboot (so the
session is resumable until then — see `ccsession`).

## Requirements

`claude` on your `PATH`.

## Install

Part of the [cctools](../../README.md) bundle:

```bash
cctools enable cchat
```

(or select it during the root `install.sh`).

## Usage

```bash
cchat                 # fresh throwaway chat
cchat --model opus    # args pass straight through to claude
cchat --help          # show help
```
```

- [ ] **Step 9: Commit**

```bash
git add tools/cchat
git commit -m "feat(cchat): import as standalone script with manifest, version, tests"
```

---

## Task 3: Import ccsession tool

ccsession is copied verbatim (script + its existing bash test suite), then wrapped with the bundle's manifest/version/changelog and a trimmed README.

**Files:**
- Create: `tools/ccsession/ccsession` (copy of `/home/paulius/source/github/plabanauskis/ccsession/ccsession`)
- Create: `tools/ccsession/tests/ccsession.test.sh` (copy)
- Create: `tools/ccsession/tool.manifest`
- Create: `tools/ccsession/VERSION`
- Create: `tools/ccsession/CHANGELOG.md`
- Create: `tools/ccsession/README.md` (trimmed)

**Interfaces:**
- Produces: command `ccsession`; entrypoint `ccsession`; manifest (`DEPS="fzf jq claude"`, `PLATFORM="linux macos"`).

- [ ] **Step 1: Copy the script and its test suite (verbatim), preserving the executable bit**

```bash
mkdir -p tools/ccsession/tests
cp /home/paulius/source/github/plabanauskis/ccsession/ccsession tools/ccsession/ccsession
cp /home/paulius/source/github/plabanauskis/ccsession/tests/ccsession.test.sh tools/ccsession/tests/ccsession.test.sh
chmod +x tools/ccsession/ccsession
```

- [ ] **Step 2: Run the copied test suite to verify the import is intact**

Run: `bash tools/ccsession/tests/ccsession.test.sh`
Expected: PASS — ends `N passed, 0 failed` (the suite sources `../ccsession`; the relative path holds in the new location).

- [ ] **Step 3: Create `tools/ccsession/tool.manifest`**

```sh
# ccsession — fzf picker to list and resume any Claude Code session.
NAME=ccsession
ENTRYPOINT=ccsession
COMMANDS="ccsession"
DEPS="fzf jq claude"
PLATFORM="linux macos"
DESC="fzf picker to list and resume any Claude Code session"
POST_ENABLE=""
```

- [ ] **Step 4: Create `tools/ccsession/VERSION`**

```
1.0.0
```

- [ ] **Step 5: Create `tools/ccsession/CHANGELOG.md`**

```markdown
# Changelog — ccsession

All notable changes to ccsession are documented here. Versions follow [semver](https://semver.org).

## 1.0.0 — 2026-06-24

- Initial release in the cctools bundle. `fzf` picker over all Claude Code
  sessions (newest-active first) with a boxed preview card; Enter `cd`s into the
  session's directory and runs `claude --resume <id>`. Dependency-free bash test
  suite included.
```

- [ ] **Step 6: Create `tools/ccsession/README.md`** (trimmed: the Install section now points at the root installer / `cctools enable`; everything else carried from the source README)

```markdown
# ccsession

List every Claude Code session in an [`fzf`](https://github.com/junegunn/fzf)
picker and resume any one of them without first navigating to its directory.

Today, resuming a session means `cd`-ing to the right project directory and
running `claude --resume`. `ccsession` replaces that with a single command: it
lists all sessions newest-active-first — status, when, directory, branch, and
summary at a glance — and on Enter `cd`s into the chosen session's original
directory and runs `claude --resume <id>` there.

## What it looks like

```
  ccsession · 47 sessions · ↑↓ select · / filter · ⏎ resume

  ●  WHEN        DIRECTORY                               BRANCH    SUMMARY
  ─  ──────────  ──────────────────────────────────────  ────────  ──────────────────────────────
  ●  4m ago      ~/source/github/plabanauskis/ccsession  —         Implement ccsession terminal tool
  ●  7h ago      ~/Downloads/k8s                         —         Create Kubernetes architecture crash course
  ✗  8h ago      ~/…/labanauskis/ccsession               main      Build terminal tool to resume sessions
  ●  8h ago      /tmp/cchat.hwzqOt                       —         Troubleshoot UniFi Network Application startup
  ●  yesterday   ~/source/github/plabanauskis/ccbox      master    Pull the latest changes
```

Highlighting a row shows a boxed preview card on the right:

```
┌─ session ──────────────────────────────────────┐
│ ● live                                         │
│                                                │
│ dir      ~/…/plabanauskis/ccsession            │
│ branch   —                                     │
│ active   2026-06-23 15:58 · 4m ago             │
│ id       e813f0d5-ae02-4685-af23-780707525c27  │
├────────────────────────────────────────────────┤
│ Implement ccsession terminal tool              │
└────────────────────────────────────────────────┘
```

## Requirements

`fzf`, `jq`, GNU `find`/`grep`/`date`/`stat`, and `claude` on your `PATH`.
(All standard on a typical Linux setup.)

## Install

Part of the [cctools](../../README.md) bundle:

```bash
cctools enable ccsession
```

(or select it during the root `install.sh`). The command is a symlink into the
bundle clone, so a `cctools update` is immediately live — no reinstall step.

## Usage

```bash
ccsession          # open the picker
ccsession --help   # show help
```

- Type to fuzzy-filter, ↑/↓ to move, Enter to resume, ESC to cancel.
- Rows marked `✗` (dimmed) are sessions whose directory no longer exists —
  e.g. ephemeral `/tmp/cchat.*` sessions after a reboot. They are shown for
  reference but cannot be resumed (pressing Enter on one prints an error and
  exits non-zero). Resuming requires the original directory because Claude
  locates sessions by their directory-derived project path.

## How it works

Sessions live at `~/.claude/projects/<encoded-dir>/<session-id>.jsonl`. For each
file `ccsession` reads:

| Field   | Source                                                       |
|---------|--------------------------------------------------------------|
| cwd     | first `"cwd"` value in the file                              |
| branch  | first `"gitBranch"` value (detached `HEAD`/none shown as `—`) |
| summary | latest `ai-title` record; else first user prompt; else id    |
| id      | the `.jsonl` filename stem                                    |
| active  | the file's mtime (drives sort order and "when")              |
| status  | `live` if the cwd still exists, else `gone`                  |

Columns are aligned by **character** count (not bytes) so the `●`/`✗`/`…`/`~`
glyphs line up under a UTF-8 locale.

## Tests

```bash
bash tests/ccsession.test.sh
```

A dependency-free bash suite that sources the script (it is written to be
sourceable) and asserts on its pure functions against synthetic session
fixtures.

## Out of scope (v1)

Deleting/renaming/archiving sessions, multi-select, opening a directory without
resuming, and searching within session contents.
```

- [ ] **Step 7: Commit**

```bash
git add tools/ccsession
git commit -m "feat(ccsession): import script + tests with manifest, version, trimmed README"
```

---

## Task 4: Import ccbox tool

ccbox is copied minus the dropped artifacts (per Global Constraints), then wrapped with the bundle manifest/version/changelog and a trimmed README. The VERSION file is seeded to **2.0.1** to match `CCBOX_VERSION` already inside `bin/ccbox`.

**Files:**
- Create: `tools/ccbox/bin/ccbox`, `tools/ccbox/Dockerfile`, `tools/ccbox/entrypoint.sh`, `tools/ccbox/test/smoke.sh`, `tools/ccbox/assets/*`, `tools/ccbox/docs/**` (all copied)
- Create: `tools/ccbox/tool.manifest`, `tools/ccbox/VERSION`, `tools/ccbox/CHANGELOG.md`
- Create: `tools/ccbox/README.md` (trimmed Install/Uninstall sections)

**Interfaces:**
- Produces: command `ccbox`; entrypoint `bin/ccbox`; manifest (`DEPS="docker claude"`, `PLATFORM="linux"`, `POST_ENABLE` build hint).

- [ ] **Step 1: Copy the kept files (preserving exec bits), dropping the rest**

```bash
mkdir -p tools/ccbox/bin tools/ccbox/test tools/ccbox/assets tools/ccbox/docs
SRC=/home/paulius/source/github/plabanauskis/ccbox
cp "$SRC/bin/ccbox"        tools/ccbox/bin/ccbox
cp "$SRC/Dockerfile"       tools/ccbox/Dockerfile
cp "$SRC/entrypoint.sh"    tools/ccbox/entrypoint.sh
cp "$SRC/test/smoke.sh"    tools/ccbox/test/smoke.sh
cp -r "$SRC/assets/."      tools/ccbox/assets/
cp -r "$SRC/docs/."        tools/ccbox/docs/
chmod +x tools/ccbox/bin/ccbox tools/ccbox/entrypoint.sh tools/ccbox/test/smoke.sh
```

Explicitly NOT copied (per Global Constraints): `packaging/`, `dist/`, `ccbox-implementation-plan.md`, `CLAUDE.md`, `LICENSE`, `.gitignore`, `.claude/`.

- [ ] **Step 2: Verify `bin/ccbox` is syntactically valid and reports its version (no Docker needed)**

Run: `bash -n tools/ccbox/bin/ccbox && bash tools/ccbox/bin/ccbox version`
Expected: no syntax error; prints `ccbox 2.0.1`.

- [ ] **Step 3: Verify the smoke test is syntactically valid (it needs Docker to run, not to parse)**

Run: `bash -n tools/ccbox/test/smoke.sh && echo OK`
Expected: `OK`.

- [ ] **Step 4: Create `tools/ccbox/tool.manifest`**

```sh
# ccbox — sandboxed autonomous Claude Code via Docker + sysbox.
NAME=ccbox
ENTRYPOINT=bin/ccbox
COMMANDS="ccbox"
DEPS="docker claude"
PLATFORM="linux"
DESC="Sandboxed autonomous Claude Code via Docker + sysbox"
POST_ENABLE="Run 'ccbox build' once to build the sandbox image (~5 min, ~5GB). sysbox-ce is checked by 'ccbox doctor'."
```

- [ ] **Step 5: Create `tools/ccbox/VERSION`** (matches `CCBOX_VERSION` in `bin/ccbox`)

```
2.0.1
```

- [ ] **Step 6: Create `tools/ccbox/CHANGELOG.md`**

```markdown
# Changelog — ccbox

All notable changes to ccbox are documented here. Versions follow [semver](https://semver.org).

## 2.0.1 — 2026-06-24

- Imported into the cctools bundle. Path-identical host mirror; runs the host's
  own `claude` binary read-only inside a sysbox container. The `.deb` packaging
  channel is dropped in favour of the bundle installer + `cctools enable ccbox`.
  Versioning moves to this file (`ccbox version` still reads `CCBOX_VERSION`).
```

- [ ] **Step 7: Create `tools/ccbox/README.md`** — carried from the source README, with the **Install** and **Uninstall** sections replaced to point at the bundle. Replace the source README's `## Install` and `## Uninstall` sections (the `.deb`/from-source blocks) with the following; keep everything else (header, Security model, Prerequisites, Usage, Troubleshooting, Architecture notes) byte-for-byte from `/home/paulius/source/github/plabanauskis/ccbox/README.md`.

New `## Install` section body:

```markdown
## Install

ccbox is part of the [cctools](../../README.md) bundle (Linux/amd64 only):

```bash
cctools enable ccbox      # symlinks the 'ccbox' command into ~/.local/bin
ccbox doctor              # check prerequisites (Docker, sysbox-ce, host claude, login)
ccbox build               # build the container image, mirroring your user (~5 min, ~5GB)
```

The `ccbox` command is a symlink into the bundle clone; `ccbox build` finds the
`Dockerfile` next to the script. The 5 GB image is built locally by `ccbox build`
(so it mirrors your username/UID/GID/home) and is never shipped.

**Toolchains baked in:** Node 24 LTS, Python 3 + `uv`, Go 1.26.x, Rust (stable), .NET 10 LTS,
plus `git`, `gh`, `jq`, `ripgrep`, `fd`, `openssl`, `socat`, and an inner Docker Engine +
Compose v2. (`claude` itself is **not** baked in — it's mounted from the host.) Versions are
`ARG`s in the `Dockerfile`; override with `--build-arg NODE_MAJOR=…` etc.

## Uninstall

```bash
ccbox uninstall          # removes the image + caches; PROMPTS before data volumes
cctools disable ccbox    # removes the 'ccbox' command symlink
```

`ccbox uninstall` never deletes your per-project database volumes (`ccbox-docker-*`)
without an explicit yes. Run it **before** `cctools uninstall` — the bundle
uninstaller never touches Docker data. (If you used ccbox 1.x, it also offers to
remove the now-unused `~/.config/ccbox` GitHub App config.)
```

(Drop the old "From the `.deb`" and "From source" subsections and the `apt remove` line entirely.)

- [ ] **Step 8: Commit**

```bash
git add tools/ccbox
git commit -m "feat(ccbox): import (minus deb/packaging/stale docs) with manifest, version, trimmed README"
```

---

## Task 5: Shared manifest lib + `bin/cctools` management command (+ tests)

The shared `lib/cctools-common.sh` (manifest-reading helpers, parameterized by `CCTOOLS_TOOLS_DIR`) plus the generic, manifest-driven lifecycle command that sources it. `bin/cctools` is sourceable for unit tests via a BASH_SOURCE guard (so strict mode + `main` run only when executed). Honors `CCTOOLS_HOME` (prefix override) and `CCTOOLS_BIN` (bindir override, for tests).

**Files:**
- Create: `lib/cctools-common.sh`
- Create: `bin/cctools`
- Test: `tests/cctools.test.sh`

**Interfaces:**
- Consumes: `tools/<name>/tool.manifest` (vars `NAME ENTRYPOINT COMMANDS DEPS PLATFORM DESC POST_ENABLE`), `tools/<name>/VERSION`.
- Produces (in `lib/cctools-common.sh`, all reading `$CCTOOLS_TOOLS_DIR`): `current_os`, `list_tools`, `load_manifest <tool>`, `platform_ok`, `missing_deps`, `tool_version <tool>`, `entrypoint_path <tool>`. These are the shared contract Task 6 (`install.sh`) also consumes.
- Produces (in `bin/cctools`): `tool_enabled`, `enable_tool <tool>`, `disable_tool <tool>`, and `cmd_*` handlers; CLI subcommands `list|doctor|enable|disable|update|uninstall|version|help`.

- [ ] **Step 1: Write the failing test** — `tests/cctools.test.sh`

```bash
#!/usr/bin/env bash
# Unit + integration tests for bin/cctools. Sources the command (BASH_SOURCE
# guard keeps main from running) and drives its functions against the real
# tools/ manifests, with an isolated bindir and a fake PATH that supplies every
# tool dep so enable/disable are testable on any host.
set -u
HERE="$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)"
REPO="$(cd "$HERE/.." && pwd)"

SANDBOX="$(mktemp -d)"
trap 'rm -rf "$SANDBOX"' EXIT
export CCTOOLS_HOME="$REPO"
export CCTOOLS_BIN="$SANDBOX/bin"
FAKEBIN="$SANDBOX/fakebin"
mkdir -p "$FAKEBIN"
for dep in claude fzf jq docker; do printf '#!/usr/bin/env bash\n' >"$FAKEBIN/$dep"; chmod +x "$FAKEBIN/$dep"; done
export PATH="$FAKEBIN:$PATH"

# shellcheck source=/dev/null
source "$REPO/bin/cctools"

PASS=0
FAIL=0
ok() { PASS=$((PASS + 1)); }
bad() { FAIL=$((FAIL + 1)); printf 'FAIL: %s\n' "$1"; }
assert_eq() { [ "$1" = "$2" ] && ok || bad "$3 (got [$1] want [$2])"; }
assert_contains() { case "$1" in *"$2"*) ok;; *) bad "$3 ([$1] lacks [$2])";; esac; }

# current_os
assert_eq "$(current_os)" "linux" "current_os is linux on this host"

# list_tools includes all three
tools="$(list_tools | sort | tr '\n' ' ')"
assert_eq "$tools" "ccbox ccsession cchat " "list_tools enumerates the three tools"

# load_manifest sets the contract vars
load_manifest cchat
assert_eq "$NAME" "cchat" "load_manifest: NAME"
assert_eq "$DEPS" "claude" "load_manifest: DEPS"
assert_contains "$PLATFORM" "linux" "load_manifest: PLATFORM"

# platform_ok: ccbox is linux-only -> ok here; (no macos-only tool to test the negative)
load_manifest ccbox
assert_eq "$(platform_ok && echo y || echo n)" "y" "platform_ok: ccbox supported on linux"

# missing_deps: empty when deps present (fake PATH), names the gap when not
load_manifest ccsession
assert_eq "$(missing_deps)" "" "missing_deps: none when fzf/jq/claude present"
assert_eq "$(PATH=/nonexistent missing_deps)" "fzf jq claude" "missing_deps: lists all when PATH empty"

# tool_version reads the VERSION file
assert_eq "$(tool_version cchat)" "1.0.0" "tool_version: cchat"
assert_eq "$(tool_version ccbox)" "2.0.1" "tool_version: ccbox"

# enable -> symlink created, tool_enabled true; disable -> removed
enable_tool cchat >/dev/null
assert_eq "$([ -L "$CCTOOLS_BIN/cchat" ] && echo y || echo n)" "y" "enable_tool: creates symlink"
assert_eq "$(readlink "$CCTOOLS_BIN/cchat")" "$REPO/tools/cchat/cchat" "enable_tool: symlink points at entrypoint"
load_manifest cchat
assert_eq "$(tool_enabled && echo y || echo n)" "y" "tool_enabled: true after enable"
disable_tool cchat >/dev/null
assert_eq "$([ -e "$CCTOOLS_BIN/cchat" ] && echo y || echo n)" "n" "disable_tool: removes symlink"

# cmd_list / cmd_version surface the tools
assert_contains "$(cmd_list)" "cchat" "cmd_list: shows cchat"
assert_contains "$(cmd_version ccbox)" "ccbox 2.0.1" "cmd_version: prints tool + version"

printf '\n%d passed, %d failed\n' "$PASS" "$FAIL"
[ "$FAIL" -eq 0 ]
```

- [ ] **Step 2: Run the test to verify it fails**

Run: `bash tests/cctools.test.sh`
Expected: FAIL (`bin/cctools` does not exist — `source: No such file`).

- [ ] **Step 3: Create `lib/cctools-common.sh`** (shared, sourced by `bin/cctools` and `install.sh`)

```bash
#!/usr/bin/env bash
# cctools-common.sh — manifest-reading helpers shared by bin/cctools and
# install.sh. Source after setting CCTOOLS_TOOLS_DIR to the bundle's tools/ dir.
# Pure function definitions only — no side effects, no strict-mode toggles — so
# it is safe to source from a strict-mode script or a test harness alike.

current_os() {
  case "$(uname -s)" in
    Linux) echo linux ;;
    Darwin) echo macos ;;
    *) echo unknown ;;
  esac
}

list_tools() {
  local d
  for d in "$CCTOOLS_TOOLS_DIR"/*/; do
    [ -f "${d}tool.manifest" ] && basename "$d"
  done
}

# Load a tool's manifest into the contract vars (reset first so stale values
# never leak between tools).
load_manifest() {
  NAME='' ENTRYPOINT='' COMMANDS='' DEPS='' PLATFORM='' DESC='' POST_ENABLE=''
  # shellcheck source=/dev/null
  . "$CCTOOLS_TOOLS_DIR/$1/tool.manifest"
}

platform_ok() { # uses $PLATFORM
  local os p
  os="$(current_os)"
  for p in $PLATFORM; do [ "$p" = "$os" ] && return 0; done
  return 1
}

missing_deps() { # uses $DEPS -> echoes space-joined missing binaries (empty if none)
  local d out=''
  for d in $DEPS; do command -v "$d" >/dev/null 2>&1 || out+="$d "; done
  printf '%s' "${out% }"
}

tool_version() { cat "$CCTOOLS_TOOLS_DIR/$1/VERSION" 2>/dev/null || echo '?'; }
entrypoint_path() { printf '%s/%s/%s' "$CCTOOLS_TOOLS_DIR" "$1" "$ENTRYPOINT"; }
```

- [ ] **Step 4: Create `bin/cctools`** (sources the lib; defines the management-specific functions)

```bash
#!/usr/bin/env bash
# cctools — manage the cctools bundle (lifecycle only; tools run by their own names).
# Sourceable for tests: strict mode + main run only when executed, not sourced.
[[ "${BASH_SOURCE[0]}" == "${0}" ]] && set -euo pipefail

SELF="$(readlink -f "${BASH_SOURCE[0]}")"
ROOT="${CCTOOLS_HOME:-$(cd "$(dirname "$SELF")/.." && pwd)}"
CCTOOLS_TOOLS_DIR="$ROOT/tools"
BIN_DIR="${CCTOOLS_BIN:-$HOME/.local/bin}"

# Shared manifest-reading helpers (current_os, list_tools, load_manifest,
# platform_ok, missing_deps, tool_version, entrypoint_path).
# shellcheck source=lib/cctools-common.sh
. "$ROOT/lib/cctools-common.sh"

tool_enabled() { # uses $NAME/$COMMANDS/$ENTRYPOINT (manifest must be loaded)
  local c link entry
  entry="$(entrypoint_path "$NAME")"
  for c in $COMMANDS; do
    link="$BIN_DIR/$c"
    [ -L "$link" ] && [ "$(readlink "$link")" = "$entry" ] || return 1
  done
  return 0
}

enable_tool() {
  load_manifest "$1"
  if ! platform_ok; then
    echo "cctools: $1 does not support $(current_os) (PLATFORM=\"$PLATFORM\")" >&2
    return 1
  fi
  local miss
  miss="$(missing_deps)"
  if [ -n "$miss" ] && [ "${FORCE:-0}" != "1" ]; then
    echo "cctools: $1 missing deps: $miss — install them, or 'cctools enable $1 --force'" >&2
    return 1
  fi
  mkdir -p "$BIN_DIR"
  local c entry
  entry="$(entrypoint_path "$1")"
  for c in $COMMANDS; do ln -sf "$entry" "$BIN_DIR/$c"; done
  echo "enabled $1 (${COMMANDS// /, } -> $BIN_DIR)"
  [ -n "$miss" ] && echo "  warning: linked despite missing deps: $miss"
  [ -n "$POST_ENABLE" ] && echo "  $POST_ENABLE"
  return 0
}

disable_tool() {
  load_manifest "$1"
  local c link entry removed=0
  entry="$(entrypoint_path "$1")"
  for c in $COMMANDS; do
    link="$BIN_DIR/$c"
    if [ -L "$link" ] && [ "$(readlink "$link")" = "$entry" ]; then
      rm -f "$link"
      removed=1
    fi
  done
  [ "$removed" = 1 ] && echo "disabled $1" || echo "$1 was not enabled"
}

cmd_list() {
  printf '%-12s %-9s %-9s %-12s %s\n' TOOL ENABLED VERSION PLATFORM DEPS
  local t state plat miss
  for t in $(list_tools); do
    load_manifest "$t"
    tool_enabled && state=enabled || state=disabled
    platform_ok && plat="ok" || plat="no:$PLATFORM"
    miss="$(missing_deps)"
    [ -z "$miss" ] && miss="ok" || miss="missing:$miss"
    printf '%-12s %-9s %-9s %-12s %s\n' "$t" "$state" "$(tool_version "$t")" "$plat" "$miss"
  done
}

cmd_doctor() {
  local tools rc=0 t d
  if [ -n "${1:-}" ]; then
    [ -f "$CCTOOLS_TOOLS_DIR/$1/tool.manifest" ] || {
      echo "cctools: unknown tool '$1'" >&2
      return 1
    }
    tools="$1"
  else
    tools="$(list_tools)"
  fi
  for t in $tools; do
    load_manifest "$t"
    echo "$t ($(tool_version "$t")) — $DESC"
    if platform_ok; then
      printf '  %-10s ok (%s)\n' platform "$(current_os)"
    else
      printf '  %-10s UNSUPPORTED (needs %s; host %s)\n' platform "$PLATFORM" "$(current_os)"
      rc=1
    fi
    for d in $DEPS; do
      if command -v "$d" >/dev/null 2>&1; then
        printf '  %-10s ok\n' "$d"
      else
        printf '  %-10s MISSING\n' "$d"
        rc=1
      fi
    done
    [ -n "$POST_ENABLE" ] && echo "  note: $POST_ENABLE"
  done
  return "$rc"
}

cmd_update() {
  command -v git >/dev/null 2>&1 || {
    echo "cctools: git not found" >&2
    return 1
  }
  echo "cctools: updating $ROOT"
  git -C "$ROOT" pull --ff-only
  local t
  for t in $(list_tools); do
    load_manifest "$t"
    if tool_enabled; then
      FORCE=1 enable_tool "$t" >/dev/null && echo "re-linked $t"
    fi
  done
  echo "cctools: update complete."
}

cmd_uninstall() {
  echo "cctools uninstall removes all command symlinks and the prefix dir:"
  echo "  prefix: $ROOT"
  echo "  bindir: $BIN_DIR"
  if command -v docker >/dev/null 2>&1; then
    if docker image inspect ccbox:latest >/dev/null 2>&1 ||
      docker volume ls -q 2>/dev/null | grep -q '^ccbox-'; then
      echo
      echo "NOTE: ccbox Docker image/volumes still exist. Run 'ccbox uninstall' FIRST"
      echo "      to remove them — cctools never deletes Docker data."
    fi
  fi
  printf 'Remove cctools now? [y/N] '
  local ans
  read -r ans </dev/tty 2>/dev/null || ans=n
  case "$ans" in y | Y) ;; *)
    echo "aborted."
    return 0
    ;;
  esac
  local t c link entry
  for t in $(list_tools); do
    load_manifest "$t"
    entry="$(entrypoint_path "$t")"
    for c in $COMMANDS; do
      link="$BIN_DIR/$c"
      [ -L "$link" ] && [ "$(readlink "$link")" = "$entry" ] && rm -f "$link" && echo "removed $link"
    done
  done
  [ -L "$BIN_DIR/cctools" ] && rm -f "$BIN_DIR/cctools" && echo "removed $BIN_DIR/cctools"
  rm -rf "$ROOT" && echo "removed $ROOT"
  echo "cctools uninstalled. Host is clean."
}

cmd_version() {
  if [ -n "${1:-}" ]; then
    [ -f "$CCTOOLS_TOOLS_DIR/$1/VERSION" ] || {
      echo "cctools: unknown tool '$1'" >&2
      return 1
    }
    echo "$1 $(tool_version "$1")"
    return 0
  fi
  local t
  for t in $(list_tools); do echo "$t $(tool_version "$t")"; done
}

usage() {
  cat <<'EOF'
cctools — manage the cctools bundle (lifecycle only; tools run by their own names)

Usage:
  cctools list                    Tools, enabled state, versions, platform + dep status
  cctools doctor [tool]           Check deps + platform for all tools or one
  cctools enable <tool> [--force] Symlink the tool's command(s) into ~/.local/bin
  cctools disable <tool>          Remove those symlinks
  cctools update                  git pull in the prefix, then re-link enabled tools
  cctools uninstall               Remove all symlinks + the prefix dir (prompts first)
  cctools version [tool]          Print version(s) from VERSION files
  cctools help                    Show this help

Prefix override: CCTOOLS_HOME (default ~/.local/share/cctools).
EOF
}

main() {
  local cmd="${1:-help}"
  shift || true
  FORCE=0
  local rest=() a
  for a in "$@"; do
    if [ "$a" = "--force" ]; then FORCE=1; else rest+=("$a"); fi
  done
  set -- "${rest[@]}"
  case "$cmd" in
    list) cmd_list ;;
    doctor) cmd_doctor "${1:-}" ;;
    enable)
      [ -n "${1:-}" ] && [ -d "$CCTOOLS_TOOLS_DIR/$1" ] || {
        echo "usage: cctools enable <tool> [--force]" >&2
        return 1
      }
      enable_tool "$1"
      ;;
    disable)
      [ -n "${1:-}" ] && [ -d "$CCTOOLS_TOOLS_DIR/$1" ] || {
        echo "usage: cctools disable <tool>" >&2
        return 1
      }
      disable_tool "$1"
      ;;
    update) cmd_update ;;
    uninstall) cmd_uninstall ;;
    version) cmd_version "${1:-}" ;;
    help | -h | --help) usage ;;
    *)
      echo "cctools: unknown command '$cmd' (try 'cctools help')" >&2
      return 1
      ;;
  esac
}

[[ "${BASH_SOURCE[0]}" == "${0}" ]] && main "$@"
```

- [ ] **Step 5: Make it executable and run the test to verify it passes**

Run: `chmod +x bin/cctools && bash tests/cctools.test.sh`
Expected: PASS — `... passed, 0 failed`.

- [ ] **Step 6: Smoke-check the real CLI surface**

Run: `CCTOOLS_HOME="$PWD" CCTOOLS_BIN="$(mktemp -d)" bash bin/cctools list && CCTOOLS_HOME="$PWD" bash bin/cctools version`
Expected: a table listing `cchat`, `ccsession`, `ccbox`; then three `name version` lines.

- [ ] **Step 7: Lint + format (both gates are enforced)**

Run: `shfmt -i 2 -ci -w lib/cctools-common.sh bin/cctools tests/cctools.test.sh && shellcheck -x lib/cctools-common.sh bin/cctools tests/cctools.test.sh && echo LINT_OK`
Expected: `LINT_OK` (no diff applied that breaks tests; no shellcheck findings). If shellcheck flags SC2154 for manifest vars in `bin/cctools` (they are assigned in the sourced lib), confirm the `# shellcheck source=lib/cctools-common.sh` directive resolves; add a narrowly-scoped `# shellcheck disable=SC2154` only if a real cross-file false positive remains. Re-run the test after any `shfmt -w` reformat.

- [ ] **Step 8: Commit**

```bash
git add lib/cctools-common.sh bin/cctools tests/cctools.test.sh
git commit -m "feat(cctools): shared manifest lib + management command + tests"
```

---

## Task 6: `install.sh` bootstrap (+ tests)

The curl|bash entrypoint. It is fetched alone, so its only pre-clone code is arg-parsing + `clone_or_update`; the manifest-reading helpers come from `lib/cctools-common.sh`, which it sources **after** cloning (the lib is guaranteed present in the freshly cloned prefix). Sourceable via `CCTOOLS_TEST_SOURCE=1` (the BASH_SOURCE guard can't be used: under `curl|bash`, `${BASH_SOURCE[0]}` is empty while `$0` is `bash`).

**Files:**
- Create: `install.sh`
- Test: `tests/install.test.sh`

**Interfaces:**
- Consumes: `lib/cctools-common.sh` (the shared helpers `current_os`, `list_tools`, `load_manifest`, `platform_ok`, `missing_deps` — Task 5), `tools/<name>/tool.manifest`, `bin/cctools` (linked unconditionally), env overrides `CCTOOLS_HOME`, `CCTOOLS_BIN`, `CCTOOLS_REPO`, `CCTOOLS_BRANCH`.
- Produces: functions `have_tty`, `load_lib` (sets `CCTOOLS_TOOLS_DIR` + sources the lib), `clone_or_update`, `link_tool`, `select_tools_auto`, `select_tools_interactive`, `print_summary`, `main`; result arrays `ENABLED SKIPPED POST`.

- [ ] **Step 1: Write the failing test** — `tests/install.test.sh`

```bash
#!/usr/bin/env bash
# Tests for install.sh: unit-test its helpers (sourced) and run one full
# non-interactive install against a file:// clone of THIS repo into a sandbox
# prefix + bindir, with a fake PATH supplying tool deps.
set -u
HERE="$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)"
REPO="$(cd "$HERE/.." && pwd)"

SANDBOX="$(mktemp -d)"
trap 'rm -rf "$SANDBOX"' EXIT
FAKEBIN="$SANDBOX/fakebin"
mkdir -p "$FAKEBIN"
for dep in claude fzf jq docker; do printf '#!/usr/bin/env bash\n' >"$FAKEBIN/$dep"; chmod +x "$FAKEBIN/$dep"; done

PASS=0
FAIL=0
ok() { PASS=$((PASS + 1)); }
bad() { FAIL=$((FAIL + 1)); printf 'FAIL: %s\n' "$1"; }
assert_eq() { [ "$1" = "$2" ] && ok || bad "$3 (got [$1] want [$2])"; }

# --- unit: source install.sh without running main, then load the shared lib
# (load_lib sets CCTOOLS_TOOLS_DIR=$CCTOOLS_HOME/tools and sources the helpers) ---
CCTOOLS_HOME="$REPO" CCTOOLS_TEST_SOURCE=1 source "$REPO/install.sh"
load_lib
assert_eq "$(current_os)" "linux" "install: current_os linux"
load_manifest cchat
assert_eq "$DEPS" "claude" "install: load_manifest reads DEPS"
assert_eq "$(platform_ok && echo y || echo n)" "y" "install: platform_ok true for cchat on linux"
assert_eq "$(PATH=/nonexistent missing_deps)" "claude" "install: missing_deps reports gap"

# --- integration: real clone + link via the actual script process ---
PREFIX="$SANDBOX/prefix"
BIN="$SANDBOX/bin"
out="$(CCTOOLS_REPO="file://$REPO" CCTOOLS_HOME="$PREFIX" CCTOOLS_BIN="$BIN" \
  PATH="$FAKEBIN:$PATH" HOME="$SANDBOX/home" \
  bash "$REPO/install.sh" --tools=cchat,ccsession 2>&1)"
assert_eq "$([ -d "$PREFIX/.git" ] && echo y || echo n)" "y" "install: clones prefix"
assert_eq "$(readlink "$BIN/cchat")" "$PREFIX/tools/cchat/cchat" "install: links cchat"
assert_eq "$(readlink "$BIN/ccsession")" "$PREFIX/tools/ccsession/ccsession" "install: links ccsession"
assert_eq "$(readlink "$BIN/cctools")" "$PREFIX/bin/cctools" "install: always links cctools"
assert_eq "$([ -e "$BIN/ccbox" ] && echo y || echo n)" "n" "install: does not link unselected ccbox"

# --- idempotent re-run (existing clone -> pull, re-link) ---
CCTOOLS_REPO="file://$REPO" CCTOOLS_HOME="$PREFIX" CCTOOLS_BIN="$BIN" \
  PATH="$FAKEBIN:$PATH" HOME="$SANDBOX/home" \
  bash "$REPO/install.sh" --tools=cchat >/dev/null 2>&1
assert_eq "$?" "0" "install: idempotent re-run succeeds"

printf '\n%d passed, %d failed\n' "$PASS" "$FAIL"
[ "$FAIL" -eq 0 ]
```

- [ ] **Step 2: Run the test to verify it fails**

Run: `bash tests/install.test.sh`
Expected: FAIL (`install.sh` missing — `source: No such file`).

- [ ] **Step 3: Create `install.sh`**

```bash
#!/usr/bin/env bash
# cctools installer — clone the bundle into a user-owned prefix and symlink
# selected tools' commands onto PATH. No sudo. Touches only $CCTOOLS_HOME and
# ~/.local/bin. Sourceable for tests via CCTOOLS_TEST_SOURCE=1.
[ "${CCTOOLS_TEST_SOURCE:-0}" = "1" ] || set -euo pipefail

REPO_URL="${CCTOOLS_REPO:-https://github.com/plabanauskis/cctools.git}"
CCTOOLS_HOME="${CCTOOLS_HOME:-$HOME/.local/share/cctools}"
BIN_DIR="${CCTOOLS_BIN:-$HOME/.local/bin}"
BRANCH="${CCTOOLS_BRANCH:-main}"

# Source the shared manifest helpers from the (already-cloned) prefix. Called
# after clone_or_update — the lib is never needed before the clone exists.
load_lib() {
  CCTOOLS_TOOLS_DIR="$CCTOOLS_HOME/tools"
  # shellcheck source=lib/cctools-common.sh
  . "$CCTOOLS_HOME/lib/cctools-common.sh"
}

have_tty() { (exec </dev/tty) 2>/dev/null; }

clone_or_update() {
  if [ -d "$CCTOOLS_HOME/.git" ]; then
    echo "cctools: updating existing clone at $CCTOOLS_HOME"
    git -C "$CCTOOLS_HOME" pull --ff-only
  else
    echo "cctools: cloning $REPO_URL -> $CCTOOLS_HOME"
    mkdir -p "$(dirname "$CCTOOLS_HOME")"
    git clone --branch "$BRANCH" "$REPO_URL" "$CCTOOLS_HOME"
  fi
}

# Link one tool's command(s); records into ENABLED / SKIPPED / POST.
link_tool() {
  load_manifest "$1"
  if ! platform_ok; then
    SKIPPED+=("$1: unsupported on $(current_os) (needs $PLATFORM); enable later: cctools enable $1")
    return 1
  fi
  local miss
  miss="$(missing_deps)"
  if [ -n "$miss" ] && [ "${FORCE:-0}" != "1" ]; then
    SKIPPED+=("$1: missing deps ($miss); install them, then: cctools enable $1")
    return 1
  fi
  mkdir -p "$BIN_DIR"
  local c entry
  entry="$CCTOOLS_HOME/tools/$1/$ENTRYPOINT"
  for c in $COMMANDS; do ln -sf "$entry" "$BIN_DIR/$c"; done
  if [ -n "$miss" ]; then ENABLED+=("$1 (forced; missing $miss)"); else ENABLED+=("$1"); fi
  [ -n "$POST_ENABLE" ] && POST+=("$1: $POST_ENABLE")
  return 0
}

# No explicit selection and no TTY: link every platform-matching, deps-clean
# tool; record skipped ones (with the command to enable each later).
select_tools_auto() {
  local t miss
  for t in $(list_tools); do
    load_manifest "$t"
    if platform_ok && [ -z "$(missing_deps)" ]; then
      SELECTED+=("$t")
    elif ! platform_ok; then
      SKIPPED+=("$t: unsupported on $(current_os) (needs $PLATFORM); enable later: cctools enable $t")
    else
      miss="$(missing_deps)"
      SKIPPED+=("$t: missing deps ($miss); install them, then: cctools enable $t")
    fi
  done
}

select_tools_interactive() {
  local t extra ans
  echo "Select tools to enable:"
  for t in $(list_tools); do
    load_manifest "$t"
    extra=''
    platform_ok || extra=" [unsupported on $(current_os)]"
    [ -n "$(missing_deps)" ] && extra="$extra [missing: $(missing_deps)]"
    printf '  enable %s — %s%s? [y/N] ' "$t" "$DESC" "$extra"
    read -r ans </dev/tty 2>/dev/null || ans=n
    case "$ans" in y | Y) SELECTED+=("$t") ;; esac
  done
}

print_summary() {
  echo
  echo "== cctools install summary =="
  if [ "${#ENABLED[@]}" -gt 0 ]; then
    echo "Enabled:"
    printf '  - %s\n' "${ENABLED[@]}"
  else
    echo "Enabled: (none)"
  fi
  if [ "${#SKIPPED[@]}" -gt 0 ]; then
    echo "Skipped:"
    printf '  - %s\n' "${SKIPPED[@]}"
  fi
  if [ "${#POST[@]}" -gt 0 ]; then
    echo "Next steps:"
    printf '  - %s\n' "${POST[@]}"
  fi
  echo "Manage with: cctools list"
  case ":$PATH:" in
    *":$BIN_DIR:"*) ;;
    *)
      echo
      echo "WARNING: $BIN_DIR is not on your PATH. Add to your shell rc:"
      echo "  export PATH=\"$BIN_DIR:\$PATH\""
      ;;
  esac
}

usage() {
  cat <<'EOF'
cctools installer

Usage:
  install.sh [--all | --tools=a,b,c] [--force]

  --all           Enable every tool (subject to platform/deps unless --force).
  --tools=LIST    Enable only the named tools (comma-separated).
  --force         Link even when a tool's deps are missing.
  (no selection)  Interactive prompt if a TTY is available, else auto-select
                  every platform-matching, deps-clean tool (skips are printed).

Env: CCTOOLS_HOME (prefix, default ~/.local/share/cctools),
     CCTOOLS_BIN (default ~/.local/bin), CCTOOLS_REPO, CCTOOLS_BRANCH.
EOF
}

main() {
  FORCE=0
  local sel_all=0 sel_csv='' a
  for a in "$@"; do
    case "$a" in
      --all) sel_all=1 ;;
      --tools=*) sel_csv="${a#--tools=}" ;;
      --force) FORCE=1 ;;
      -h | --help)
        usage
        return 0
        ;;
      *)
        echo "install.sh: unknown arg '$a'" >&2
        usage
        return 1
        ;;
    esac
  done

  command -v git >/dev/null 2>&1 || {
    echo "install.sh: git is required" >&2
    return 1
  }
  echo "cctools installer — host $(uname -s)/$(uname -m)"
  clone_or_update
  load_lib # shared manifest helpers now available (current_os, list_tools, ...)

  SELECTED=()
  ENABLED=()
  SKIPPED=()
  POST=()
  if [ "$sel_all" = 1 ]; then
    local t
    for t in $(list_tools); do SELECTED+=("$t"); done
  elif [ -n "$sel_csv" ]; then
    local IFS=, t
    for t in $sel_csv; do
      if [ -d "$CCTOOLS_HOME/tools/$t" ]; then SELECTED+=("$t"); else echo "install.sh: unknown tool '$t' (ignored)" >&2; fi
    done
    unset IFS
  elif have_tty; then
    select_tools_interactive
  else
    select_tools_auto
  fi

  local t
  for t in "${SELECTED[@]}"; do link_tool "$t" || true; done

  mkdir -p "$BIN_DIR"
  ln -sf "$CCTOOLS_HOME/bin/cctools" "$BIN_DIR/cctools"

  print_summary
}

[ "${CCTOOLS_TEST_SOURCE:-0}" = "1" ] || main "$@"
```

- [ ] **Step 4: Run the test to verify it passes**

Run: `bash tests/install.test.sh`
Expected: PASS — `... passed, 0 failed`. (The integration step clones the committed tools + `lib/` + `bin/cctools` from tasks 2–5; `bin/cctools` is what the `cctools` symlink targets, and `lib/cctools-common.sh` is what `load_lib` sources from the clone.)

- [ ] **Step 5: Lint + format (both gates enforced)**

Run: `shfmt -i 2 -ci -w install.sh tests/install.test.sh && shellcheck -x install.sh tests/install.test.sh && echo LINT_OK`
Expected: `LINT_OK`. The `# shellcheck source=lib/cctools-common.sh` directive lets `-x` resolve the helpers; if a residual SC2154 false positive remains for a manifest var, add a narrowly-scoped disable. Re-run the test after any reformat.

- [ ] **Step 6: Commit**

```bash
git add install.sh tests/install.test.sh
git commit -m "feat(install): manifest-driven curl|bash bootstrap + tests"
```

---

## Task 7: `scripts/check.sh` (local CI)

Aggregates lint + format + every test suite. shellcheck/shfmt run only if installed (loud SKIP otherwise — never a silent pass). ccbox smoke runs only when sysbox AND the image are present.

**Files:**
- Create: `scripts/check.sh`

**Interfaces:**
- Consumes: every script + every `tests/`/`test/` suite in the repo.
- Produces: exit 0 (all gates pass/skip cleanly) or non-zero (any failure).

- [ ] **Step 1: Create `scripts/check.sh`**

```bash
#!/usr/bin/env bash
# Local "CI": shellcheck + shfmt (run-if-present) + all test suites + gated ccbox
# smoke. Run from anywhere; resolves the repo root itself. Never a silent pass:
# a missing linter prints SKIP with an install hint, not OK.
set -uo pipefail

ROOT="$(cd "$(dirname "${BASH_SOURCE[0]}")/.." && pwd)"
cd "$ROOT"
SHFMT_FLAGS=(-i 2 -ci)
rc=0

# All shell scripts we own (imported tool scripts included).
mapfile -t SCRIPTS < <(
  printf '%s\n' \
    install.sh \
    lib/cctools-common.sh \
    bin/cctools \
    scripts/check.sh scripts/dev-setup.sh scripts/release.sh \
    .githooks/pre-push \
    tools/cchat/cchat \
    tools/ccsession/ccsession \
    tools/ccbox/bin/ccbox tools/ccbox/entrypoint.sh tools/ccbox/test/smoke.sh
  find tests tools/*/tests -name '*.test.sh' -type f 2>/dev/null
)

echo "== bash -n (syntax) =="
for f in "${SCRIPTS[@]}"; do
  [ -f "$f" ] || continue
  if bash -n "$f"; then echo "  ok   $f"; else
    echo "  FAIL $f"
    rc=1
  fi
done

echo "== shellcheck =="
if command -v shellcheck >/dev/null 2>&1; then
  for f in "${SCRIPTS[@]}"; do
    [ -f "$f" ] || continue
    if shellcheck -x "$f"; then echo "  ok   $f"; else
      echo "  FAIL $f"
      rc=1
    fi
  done
else
  echo "  SKIP: shellcheck not installed (apt install shellcheck / brew install shellcheck)"
fi

echo "== shfmt --diff =="
if command -v shfmt >/dev/null 2>&1; then
  for f in "${SCRIPTS[@]}"; do
    [ -f "$f" ] || continue
    if shfmt "${SHFMT_FLAGS[@]}" -d "$f" | grep -q .; then
      echo "  FAIL $f (run: shfmt ${SHFMT_FLAGS[*]} -w $f)"
      shfmt "${SHFMT_FLAGS[@]}" -d "$f"
      rc=1
    else
      echo "  ok   $f"
    fi
  done
else
  echo "  SKIP: shfmt not installed (go install mvdan.cc/sh/v3/cmd/shfmt@latest)"
fi

echo "== test suites =="
while IFS= read -r t; do
  [ -f "$t" ] || continue
  echo "--- $t"
  if bash "$t"; then echo "  PASS $t"; else
    echo "  FAIL $t"
    rc=1
  fi
done < <(find tests tools/*/tests -name '*.test.sh' -type f 2>/dev/null | sort)

echo "== ccbox smoke =="
if docker info -f '{{.Runtimes}}' 2>/dev/null | grep -q sysbox-runc; then
  if docker image inspect "${CCBOX_IMAGE:-ccbox:latest}" >/dev/null 2>&1; then
    if bash tools/ccbox/test/smoke.sh "${CCBOX_IMAGE:-ccbox:latest}"; then echo "  PASS"; else
      echo "  FAIL"
      rc=1
    fi
  else
    echo "  SKIP: ccbox image not built (run 'ccbox build')"
  fi
else
  echo "  SKIP: ccbox smoke (no sysbox)"
fi

echo
[ "$rc" -eq 0 ] && echo "check.sh: ALL CHECKS PASSED" || echo "check.sh: SOME CHECKS FAILED"
exit "$rc"
```

- [ ] **Step 2: Make it executable and run it**

Run: `chmod +x scripts/check.sh && scripts/check.sh`
Expected: `bash -n` all `ok`; shellcheck all `ok` and shfmt all `ok` (both are installed, so both gates run and must pass — fix any finding before continuing); all test suites `PASS`; `ccbox smoke` `SKIP: ccbox image not built` (sysbox present, image absent); final `check.sh: ALL CHECKS PASSED`, exit 0.

- [ ] **Step 3: Commit**

```bash
git add scripts/check.sh
git commit -m "feat(scripts): add check.sh local CI (lint + format + tests + gated smoke)"
```

---

## Task 8: `.githooks/pre-push` + `scripts/dev-setup.sh`

**Files:**
- Create: `.githooks/pre-push`
- Create: `scripts/dev-setup.sh`

**Interfaces:**
- Consumes: `scripts/check.sh`.
- Produces: a push gate; a one-shot hook installer.

- [ ] **Step 1: Create `.githooks/pre-push`**

```bash
#!/usr/bin/env bash
# Blocks 'git push' if local checks fail. Enable with: scripts/dev-setup.sh
set -euo pipefail
exec "$(git rev-parse --show-toplevel)/scripts/check.sh"
```

- [ ] **Step 2: Create `scripts/dev-setup.sh`**

```bash
#!/usr/bin/env bash
# One-time: point git at the repo's hooks so 'git push' runs scripts/check.sh.
set -euo pipefail
cd "$(git rev-parse --show-toplevel)"
git config core.hooksPath .githooks
echo "Dev hooks enabled (core.hooksPath=.githooks)."
echo "scripts/check.sh will now run on every 'git push' and block it on failure."
```

- [ ] **Step 3: Make both executable and verify the hook resolves check.sh**

Run: `chmod +x .githooks/pre-push scripts/dev-setup.sh && bash -n .githooks/pre-push && bash -n scripts/dev-setup.sh && echo OK`
Expected: `OK`.

- [ ] **Step 4: Wire up the hook for this repo and confirm**

Run: `scripts/dev-setup.sh && git config --get core.hooksPath`
Expected: prints the enablement message, then `.githooks`.

- [ ] **Step 5: Commit**

```bash
git add .githooks/pre-push scripts/dev-setup.sh
git commit -m "feat(scripts): add pre-push hook + dev-setup (core.hooksPath)"
```

---

## Task 9: `scripts/release.sh` (per-tool release helper)

Optional helper that bumps a tool's `VERSION` + `CHANGELOG.md`, commits, tags `<tool>-vX.Y.Z`, and (with `--gh`) cuts a GitHub release. The bump functions are sourceable for testing; git/tag/gh side effects run only in `main`. For ccbox, also syncs `CCBOX_VERSION` inside `bin/ccbox` so `ccbox version` stays correct.

**Files:**
- Create: `scripts/release.sh`
- Test: `tests/release.test.sh`

**Interfaces:**
- Consumes: `tools/<tool>/VERSION`, `tools/<tool>/CHANGELOG.md`, (ccbox) `tools/ccbox/bin/ccbox`.
- Produces: functions `is_semver <v>`, `bump_version <tool> <ver>`, `prepend_changelog <tool> <ver> <date>`; CLI `release.sh <tool> <version> [--gh] [--date=YYYY-MM-DD]`.

- [ ] **Step 1: Write the failing test** — `tests/release.test.sh`

```bash
#!/usr/bin/env bash
# Tests for release.sh bump helpers. Sources the script (guard keeps main from
# running) and exercises bump_version/prepend_changelog against a temp copy of
# a tool dir, so no real commit/tag happens.
set -u
HERE="$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)"
REPO="$(cd "$HERE/.." && pwd)"

SANDBOX="$(mktemp -d)"
trap 'rm -rf "$SANDBOX"' EXIT
# Stage a fake repo root with one tool dir.
export CCTOOLS_HOME="$SANDBOX/repo"
mkdir -p "$CCTOOLS_HOME/tools/cchat"
echo "1.0.0" >"$CCTOOLS_HOME/tools/cchat/VERSION"
printf '# Changelog — cchat\n\n## 1.0.0 — 2026-06-24\n\n- Initial.\n' >"$CCTOOLS_HOME/tools/cchat/CHANGELOG.md"

# shellcheck source=/dev/null
source "$REPO/scripts/release.sh"

PASS=0
FAIL=0
ok() { PASS=$((PASS + 1)); }
bad() { FAIL=$((FAIL + 1)); printf 'FAIL: %s\n' "$1"; }
assert_eq() { [ "$1" = "$2" ] && ok || bad "$3 (got [$1] want [$2])"; }

assert_eq "$(is_semver 1.2.3 && echo y || echo n)" "y" "is_semver: accepts 1.2.3"
assert_eq "$(is_semver v1.2 && echo y || echo n)" "n" "is_semver: rejects v1.2"

bump_version cchat 1.1.0
assert_eq "$(cat "$CCTOOLS_HOME/tools/cchat/VERSION")" "1.1.0" "bump_version: writes new version"

prepend_changelog cchat 1.1.0 2026-07-01
head1="$(head -n1 "$CCTOOLS_HOME/tools/cchat/CHANGELOG.md")"
assert_eq "$head1" "# Changelog — cchat" "prepend_changelog: keeps title first"
grep -q "## 1.1.0 — 2026-07-01" "$CCTOOLS_HOME/tools/cchat/CHANGELOG.md" && ok || bad "prepend_changelog: adds new heading"
grep -q "## 1.0.0 — 2026-06-24" "$CCTOOLS_HOME/tools/cchat/CHANGELOG.md" && ok || bad "prepend_changelog: keeps old entry"

printf '\n%d passed, %d failed\n' "$PASS" "$FAIL"
[ "$FAIL" -eq 0 ]
```

- [ ] **Step 2: Run the test to verify it fails**

Run: `bash tests/release.test.sh`
Expected: FAIL (`scripts/release.sh` missing — `source: No such file`).

- [ ] **Step 3: Create `scripts/release.sh`**

```bash
#!/usr/bin/env bash
# Per-tool release helper: bump VERSION + CHANGELOG, commit, tag <tool>-vX.Y.Z,
# and (with --gh) cut a GitHub release. Bump fns are sourceable for tests; git
# side effects run only in main. Honors CCTOOLS_HOME (default = repo root).
[[ "${BASH_SOURCE[0]}" == "${0}" ]] && set -euo pipefail

RELEASE_ROOT="${CCTOOLS_HOME:-$(cd "$(dirname "${BASH_SOURCE[0]}")/.." && pwd)}"

is_semver() { [[ "$1" =~ ^[0-9]+\.[0-9]+\.[0-9]+$ ]]; }

bump_version() { # <tool> <version>
  echo "$2" >"$RELEASE_ROOT/tools/$1/VERSION"
  # Keep ccbox's embedded CCBOX_VERSION in sync so 'ccbox version' matches.
  if [ "$1" = "ccbox" ] && [ -f "$RELEASE_ROOT/tools/ccbox/bin/ccbox" ]; then
    sed -i -E "s/^CCBOX_VERSION=\"[^\"]*\"/CCBOX_VERSION=\"$2\"/" \
      "$RELEASE_ROOT/tools/ccbox/bin/ccbox"
  fi
}

prepend_changelog() { # <tool> <version> <date>
  local cl="$RELEASE_ROOT/tools/$1/CHANGELOG.md" tmp
  tmp="$(mktemp)"
  # Title line stays first; insert the new entry directly beneath it.
  {
    head -n1 "$cl"
    printf '\n## %s — %s\n\n- TODO: summarize changes.\n' "$2" "$3"
    tail -n +2 "$cl"
  } >"$tmp"
  mv "$tmp" "$cl"
}

usage() {
  cat <<'EOF'
release.sh — cut a per-tool release

Usage:
  scripts/release.sh <tool> <version> [--gh] [--date=YYYY-MM-DD]

Bumps tools/<tool>/VERSION + CHANGELOG.md, commits, and tags <tool>-vX.Y.Z.
--gh also runs 'gh release create <tool>-vX.Y.Z'. Edit the CHANGELOG's TODO line
before pushing.
EOF
}

main() {
  local tool="${1:-}" version="${2:-}" do_gh=0 date_str=''
  shift 2 2>/dev/null || {
    usage
    return 1
  }
  local a
  for a in "$@"; do
    case "$a" in
      --gh) do_gh=1 ;;
      --date=*) date_str="${a#--date=}" ;;
      *)
        echo "release.sh: unknown arg '$a'" >&2
        return 1
        ;;
    esac
  done
  [ -d "$RELEASE_ROOT/tools/$tool" ] || {
    echo "release.sh: unknown tool '$tool'" >&2
    return 1
  }
  is_semver "$version" || {
    echo "release.sh: '$version' is not X.Y.Z" >&2
    return 1
  }
  [ -n "$date_str" ] || date_str="$(date +%Y-%m-%d)"

  bump_version "$tool" "$version"
  prepend_changelog "$tool" "$version" "$date_str"
  echo "Bumped $tool -> $version (edit the CHANGELOG TODO line, then continue)."

  local tag="$tool-v$version"
  git -C "$RELEASE_ROOT" add "tools/$tool/VERSION" "tools/$tool/CHANGELOG.md"
  [ "$tool" = "ccbox" ] && git -C "$RELEASE_ROOT" add "tools/ccbox/bin/ccbox"
  git -C "$RELEASE_ROOT" commit -m "release($tool): $version"
  git -C "$RELEASE_ROOT" tag -a "$tag" -m "$tool $version"
  echo "Committed and tagged $tag. Push with: git push origin HEAD $tag"

  if [ "$do_gh" = 1 ]; then
    command -v gh >/dev/null 2>&1 || {
      echo "release.sh: gh not installed" >&2
      return 1
    }
    gh release create "$tag" --title "$tool $version" \
      --notes "See tools/$tool/CHANGELOG.md"
  fi
}

[[ "${BASH_SOURCE[0]}" == "${0}" ]] && main "$@"
```

- [ ] **Step 4: Make it executable and run the test to verify it passes**

Run: `chmod +x scripts/release.sh && bash tests/release.test.sh`
Expected: PASS — `... passed, 0 failed`.

- [ ] **Step 5: Commit**

```bash
git add scripts/release.sh tests/release.test.sh
git commit -m "feat(scripts): add per-tool release.sh helper + tests"
```

---

## Task 10: Root `README.md` (front door)

**Files:**
- Create: `README.md`

**Interfaces:**
- Consumes: nothing programmatically; links to per-tool READMEs + ccbox threat model.

- [ ] **Step 1: Create `README.md`**

````markdown
# cctools

Three small, sharp helpers for working with [Claude Code](https://claude.com/claude-code)
in the Linux/macOS terminal — bundled behind one installer and a tiny management
command. Install only what you want; uninstall leaves your machine exactly as it was.

| Tool | What it does | Deps | Platform |
|------|--------------|------|----------|
| **cchat** | Opens Claude Code in a fresh ephemeral `/tmp` dir for throwaway chats | `claude` | linux, macos |
| **ccsession** | `fzf` picker to list and resume any Claude Code session without `cd`-ing | `fzf`, `jq`, `claude` | linux, macos |
| **ccbox** | Sandboxed autonomous Claude Code via Docker + sysbox (path-identical host mirror) | `docker`, `sysbox-ce`, `claude` | linux (amd64) |

cchat and ccsession are featherweight and portable; ccbox is heavy (a ~5 GB Docker
image) and Linux-only. The installer is dep-aware and opt-in, so wanting `ccsession`
never drags you into ccbox's Docker world.

## Install

```bash
curl -fsSL https://raw.githubusercontent.com/plabanauskis/cctools/main/install.sh | bash
```

Prefer to read before you run (recommended):

```bash
curl -fsSL -O https://raw.githubusercontent.com/plabanauskis/cctools/main/install.sh
less install.sh
bash install.sh            # interactive picker
bash install.sh --all      # or: --tools=cchat,ccsession   /   --force
```

## What gets installed (and how to remove it)

No sudo. Nothing in `/usr` or `/etc`; no daemons. The installer touches **only two
user-owned locations** (the same pattern as `rustup`, `nvm`, and Claude Code's own
installer):

- `~/.local/share/cctools` — a git clone of this repo (the single source of truth).
  Override with `CCTOOLS_HOME`.
- `~/.local/bin` — one symlink per enabled command, plus `cctools`.

To remove everything:

```bash
ccbox uninstall      # FIRST, only if you used ccbox — removes its image/volumes
cctools uninstall    # removes the symlinks + the clone; prompts first
```

Because nothing else was ever created, your host is pristine afterward.

## Managing tools

Tools run by their own names (`cchat`, `ccsession`, `ccbox`). `cctools` is for
lifecycle only:

```bash
cctools list                 # tools, enabled state, versions, platform + dep status
cctools doctor [tool]        # check deps + platform
cctools enable <tool>        # symlink the tool's command(s) into ~/.local/bin
cctools disable <tool>       # remove them
cctools update               # git pull the clone, then re-link enabled tools
cctools version [tool]       # versions from VERSION files
```

If `~/.local/bin` isn't on your `PATH`, the installer prints the exact line to add.

## The tools

- [cchat](tools/cchat/README.md) — throwaway chats in a fresh temp dir.
- [ccsession](tools/ccsession/README.md) — `fzf` session picker + resume.
- [ccbox](tools/ccbox/README.md) — sandboxed autonomous Claude Code.

## Security

ccbox runs Claude Code with `--dangerously-skip-permissions` inside a sysbox
container that mirrors your environment but isolates the system layer. Read its
[security model and threat model](tools/ccbox/README.md#security-model) before
using it. cchat and ccsession run no privileged operations.

## Development

```bash
scripts/dev-setup.sh   # wire up the pre-push hook (runs scripts/check.sh)
scripts/check.sh       # shellcheck + shfmt + all tool tests + gated ccbox smoke
```

Per-tool releases: `scripts/release.sh <tool> <version>` (tags `<tool>-vX.Y.Z`).

## License

[MIT](LICENSE). Portions of ccbox adapted from
[RchGrav/claudebox](https://github.com/RchGrav/claudebox) (MIT).
````

- [ ] **Step 2: Verify the per-tool links resolve to real files**

Run: `for f in tools/cchat/README.md tools/ccsession/README.md tools/ccbox/README.md LICENSE; do test -f "$f" && echo "ok $f" || echo "MISSING $f"; done`
Expected: four `ok` lines.

- [ ] **Step 3: Commit**

```bash
git add README.md
git commit -m "docs: add root README front door (pitch, install, matrix, uninstall promise)"
```

---

## Task 11: Final verification & local migration

Run the full local CI, then perform the environment migration the spec calls for. The two host-affecting steps (editing `~/.zshrc`, publishing to GitHub) require explicit user confirmation at execution time — do not perform them silently.

**Files:**
- Modify (with confirmation): `~/.zshrc` (remove the `cchat` function)

**Interfaces:**
- Consumes: the whole repo.

- [ ] **Step 1: Run the full check suite**

Run: `scripts/check.sh`
Expected: `check.sh: ALL CHECKS PASSED`, exit 0 (shellcheck/shfmt SKIP on this machine; ccbox smoke SKIP — image not built).

- [ ] **Step 2: Verify the whole tree is committed and the layout matches the spec**

Run: `git status --porcelain && echo '---' && git ls-files | sed 's#/.*##' | sort -u`
Expected: clean working tree; top-level entries include `LICENSE README.md install.sh bin docs scripts tests tools .githooks .gitignore`.

- [ ] **Step 3: (Confirm with user) Remove the `cchat` function from `~/.zshrc`**

A shell function shadows a PATH command, so the old `cchat()` in `~/.zshrc` (lines ~123–128) must go for the new `cchat` script to win. After confirming with the user, delete exactly the comment + function block:

```
# cchat — quick throwaway chat with Claude Code in a fresh /tmp dir
cchat() {
  local dir
  dir=$(mktemp -d "/tmp/cchat.XXXXXX") || return
  (cd "$dir" && claude "$@")
}
```

Verify afterward: `grep -n 'cchat' ~/.zshrc` should show no function definition (only unrelated matches, if any).

- [ ] **Step 4: (Confirm with user) Install locally so the commands land on PATH**

From a local clone (no GitHub needed):

```bash
CCTOOLS_REPO="file://$PWD" bash install.sh --tools=cchat,ccsession
# ccbox is Linux/amd64 + Docker/sysbox; enable when ready:
#   cctools enable ccbox && ccbox build
```

Ensure `~/.local/bin` is on `PATH` (the installer warns + prints the line if not).
Verify: `cctools list` shows the enabled tools.

- [ ] **Step 5: (Confirm with user) Publish to GitHub**

Outward-facing — do only with explicit user approval:

```bash
gh repo create plabanauskis/cctools --private --source=. --remote=origin --push
```

Leave the repo PRIVATE; the maintainer flips it public and deletes the legacy
`plabanauskis/ccbox` + `plabanauskis/ccsession` repos manually when ready.

---

## Self-Review

**1. Spec coverage:**
- §1 purpose / tool matrix → README (Task 10), manifests (Tasks 2–4). ✓
- §2 repo layout → all tasks; `tests/` at root is an additive extension (TDD of plumbing), noted in Global Constraints + File Structure. ✓
- §3 manifest contract → manifests (Tasks 2–4) + the single shared reader `lib/cctools-common.sh` (Task 5) used by both `bin/cctools` and `install.sh` (Task 6). ✓
- §4 install model (no sudo, two locations, select tools, dep/platform handling, PATH warn, summary, reversible) → Task 6. ✓
- §5 CLI surface (`cctools` management-only; cchat standalone script) → Tasks 5, 2. ✓
- §6 versioning/releases (VERSION+CHANGELOG, `<tool>-vX.Y.Z`, release.sh) → Tasks 2–4, 9. ✓
- §7 local checks (check.sh = shellcheck + shfmt — both installed, so ENFORCED — + tests; pre-push; dev-setup; ccbox smoke gated on sysbox+image) → Tasks 7, 8. ✓
- §8 docs (root README front door, per-tool README trims) → Tasks 10, 3, 4. ✓
- §9 migration/publish (single import = copy trees not history; remove cchat from zshrc; private repo) → Tasks 2–4 (copy, no history), Task 11. ✓
- §10 phases → task ordering 1→11. ✓
- "Out of scope" items → none built (constraints list them). ✓

**2. Placeholder scan:** No TBD/TODO in shipped code. The only literal "TODO" is the CHANGELOG stub line that `release.sh` intentionally writes for the human to fill at release time (documented in the helptext + commit flow) — not a plan placeholder.

**3. Type/name consistency:** Manifest var names (`NAME ENTRYPOINT COMMANDS DEPS PLATFORM DESC POST_ENABLE`) are identical across all manifests and the shared `lib/cctools-common.sh`. The helpers now live in ONE place (the lib); `bin/cctools` sets `CCTOOLS_TOOLS_DIR="$ROOT/tools"` and `install.sh`'s `load_lib` sets `CCTOOLS_TOOLS_DIR="$CCTOOLS_HOME/tools"` before sourcing — the lib reads `$CCTOOLS_TOOLS_DIR` consistently. Env overrides (`CCTOOLS_HOME`, `CCTOOLS_BIN`, `CCTOOLS_REPO`) are spelled consistently in scripts and tests. ccbox version is `2.0.1` everywhere (VERSION file, `CCBOX_VERSION`, CHANGELOG, tests, release sync).

**Deviations noted for reviewer:**
- spec §6 says "ccbox 2.0.0" but `bin/ccbox` already carries `CCBOX_VERSION="2.0.1"`; the plan seeds VERSION to **2.0.1** so `cctools version ccbox` equals `ccbox version` (honoring the spec's stated intent: "seeded to current reality").
- The §2 layout shows no `lib/`; the plan adds `lib/cctools-common.sh` (approved with the maintainer) so the shared manifest helpers live in one place instead of being duplicated between `bin/cctools` and `install.sh` — serving the spec's "generic, manifest-driven, thin bootstrap" intent.
