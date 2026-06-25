# cctools tool logos & docs polish — design

**Date:** 2026-06-25
**Status:** approved (pre-implementation)

## Goal

Give cctools a "ready for consumption" face before the repo goes public, by giving
each of the three tools its own logo in a single coherent visual family, and wiring
those logos into the docs. Take the existing `ccbox` brand as the anchor and extend
it to `cchat` and `ccsession`.

No separate umbrella/`cctools` mark — the brand *is* the three tools as a family.

## Scope

In scope:

- A logo + icon + dark variants + social card for **cchat** and **ccsession**, matching
  the asset set `ccbox` already ships.
- README/docs polish: each logo in its tool's README header, and a branded tool-row in
  the top-level `README.md`.

Out of scope (explicitly, this pass):

- A standalone `cctools` umbrella logo.
- A landing page / GitHub Pages site.
- OSS-launch hygiene files (CONTRIBUTING, SECURITY.md, issue/PR templates, CI workflow).
- Any change to `ccbox`'s existing assets — reused unchanged.

## The shared system (held identical across all three)

Anchored on the existing `ccbox` assets in `tools/ccbox/assets/`:

- **Palette:** `#D97757` clay (the accent), `#16181D` near-black, `#F4F1EA` warm cream
  (used as the foreground in `-dark` variants).
- **Wordmark:** monospace (`ui-monospace, 'JetBrains Mono', …`), `font-weight:700`,
  `letter-spacing:-1`; the `cc` prefix in clay, the rest of the name in dark
  (cream in `-dark` variants). i.e. `cc` clay + suffix dark.
- **Family thread:** the clay `>` prompt chevron (the existing ccbox polyline) appears
  inside every tool's glyph — "Claude Code, framed differently."
- **Style:** flat, geometric, single-accent. No gradients, no extra colors.

## The three marks

| Tool | Glyph (the "container") | Meaning |
|---|---|---|
| **ccbox** | isometric **cube** (clay top, dark faces) with `>` on the left face | the sandbox — unchanged |
| **cchat** | **speech bubble** (clay outline) with the `>` prompt inside | the throwaway conversation |
| **ccsession** | **picker rows** — a short stack of horizontal bars, one highlighted in clay with the `>` as its selection cursor | the fzf session picker |

The glyphs share the cube's footprint/scale and sit to the left of the wordmark in the
full `logo` lockups, exactly as ccbox does (icon ≈ 128×128 viewBox region; logo ≈ 360×160).

## Asset set per tool (mirrors ccbox)

For both `cchat` and `ccsession`, under `tools/<tool>/assets/`:

- `logo.svg` — glyph + wordmark, light background (dark text)
- `logo-dark.svg` — glyph + wordmark, dark background (cream text)
- `icon.svg` — glyph only, light
- `icon-dark.svg` — glyph only, dark
- `og-image.svg` — 1280×640 social card (dark bg, faint oversized glyph motif, centered
  glyph + wordmark + two-line tagline + repo footer), matching ccbox's `og-image.svg`
- `og-image.png` — rasterized from the SVG at 1280×640

### Taglines (for the og cards — refinable)

- **cchat:** "A throwaway Claude Code chat, in one command. Leaves no trace."
- **ccsession:** "Find and resume any Claude Code session — no `cd` required."
- **ccbox** (existing, for reference): "Give Claude Code full control of your project,
  never of your computer."

The og footer reads `github.com/plabanauskis/cctools` for both (the suite repo), unlike
ccbox's standalone-repo footer.

## PNG rasterization

`og-image.png` must be generated from the SVG. Resolve the available rasterizer at
implementation time, in this order of preference: `rsvg-convert` → `inkscape` →
headless Chromium (Playwright). Render at exactly 1280×640. If no rasterizer is
available, ship the `.svg` and flag the missing `.png` rather than committing a broken
binary.

## Docs changes

- `tools/cchat/README.md` and `tools/ccsession/README.md`: add a logo header at the top
  (picture element with `logo-dark.svg`/`logo.svg` for dark/light, as is conventional),
  matching whatever header convention ccbox's README uses (or establishing it if ccbox
  has none yet).
- Top-level `README.md`: add a branded header and render the three tools with their
  icons in the existing tool table / tool list, so the suite reads as one set.
- Keep all existing prose; this is additive polish, not a rewrite.

## Verification

- All SVGs are well-formed XML and render in a browser without errors.
- Each `og-image.png` is exactly 1280×640.
- The three icons are visually distinguishable at 32px yet obviously the same family.
- READMEs render correctly on GitHub (relative asset paths resolve; `<picture>` dark/light
  switch works).
- `scripts/check.sh` still passes (no shell/test regressions — assets are inert, but the
  run confirms nothing was broken).
