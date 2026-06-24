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
