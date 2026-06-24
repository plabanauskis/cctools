# Changelog — ccbox

All notable changes to ccbox are documented here. Versions follow [semver](https://semver.org).

## 2.0.1 — 2026-06-24

- Imported into the cctools bundle. Path-identical host mirror; runs the host's
  own `claude` binary read-only inside a sysbox container. The `.deb` packaging
  channel is dropped in favour of the bundle installer + `cctools enable ccbox`.
  Versioning moves to this file (`ccbox version` still reads `CCBOX_VERSION`).
