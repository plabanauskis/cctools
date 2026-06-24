# Changelog — ccbox

All notable changes to ccbox are documented here. Versions follow [semver](https://semver.org).

## 1.0.0 — 2026-06-24

- Initial release in the cctools bundle (fresh `1.0.0` baseline alongside cchat
  and ccsession). Sandboxed autonomous Claude Code: a path-identical host mirror
  that runs the host's own `claude` binary read-only inside a sysbox container.
- Distribution moves from a standalone `.deb` to the bundle installer +
  `cctools enable ccbox`; `ccbox uninstall` teardown now points at
  `cctools disable ccbox`, and `ccbox version` reads `CCBOX_VERSION` from
  `bin/ccbox` (kept in sync with this tool's `VERSION` file).
