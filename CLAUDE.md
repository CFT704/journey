# CLAUDE.md

Guidance for AI assistants working in this repository.

## What this is

**Jimmy's Journey** — a single-page sobriety tracker built as a personal gift for
one user ("Dad", a 65-year-old, on an iPhone). It counts days since day zero,
walks a physical-healing timeline, tallies money not spent on alcohol, and tracks
"sober firsts" (birthdays/anniversaries met sober).

It is designed to be installed via Safari → Share → **Add to Home Screen** and to
work forever, offline, with no server, no build step, and no dependencies.

## Repository layout

The entire app is four files at the repo root. There is no `src/`, no
`package.json`, no lockfile, no CI.

| File | Purpose |
| --- | --- |
| `index.html` | The whole application: markup, CSS, JS, and base64-embedded fonts + icons. |
| `icon.png` | 512×512 home-screen icon (also embedded twice inside `index.html`). |
| `.nojekyll` | Tells GitHub Pages to serve the tree verbatim (no Jekyll processing). |
| `CLAUDE.md` | This file. |

Deployment is static hosting of the repo root — push to `main` and the site is the
site. There is nothing to build or bundle.

## Working with `index.html`

The file is ~560 lines but ~185 KB, because two `@font-face` rules and two icon
`<link>`s carry base64 payloads on single enormous lines.

**Do not `Read` the whole file** — it blows the token limit. Read by range
instead. Structure as of this writing:

| Lines | Contents |
| --- | --- |
| 1–12 | `<head>` meta: viewport with `viewport-fit=cover`, `apple-mobile-web-app-*`, `theme-color`, `apple-touch-icon` |
| 13–14 | Base64 icon data URIs (~26 KB each) — **skip when reading** |
| 15–18 | `<style>` open + two Lora `@font-face` blocks (~50 KB each) — **skip when reading** |
| 19–83 | Design tokens (`:root` custom properties), base element styles, `.btn`/`.input`/`.card` primitives, keyframes, `.unit*` counter styles |
| 85–196 | Body markup: JS-warning banner, hero section, healing timeline mount, money card, occasions section, install hint, mailto footer |
| 197–558 | The single `<script>` IIFE |

To read code without the blobs: `sed -n '19,83p' index.html`, `sed -n '85,196p'`,
`sed -n '197,560p'`. To inspect a blob line safely, pipe through `cut -c1-120`.

### Script organization (inside one `'use strict'` IIFE)

1. **Constants** — `DAY`, `QUIT` (day zero), feature flags `SHOW_EXTENDED` and `FLOURISH`.
2. **Persisted state** — read from `localStorage` inside `try`/`catch`.
3. **Milestones** — `steps` array plus yearly steps through year 50; `ord()`, `name()`, `money()` formatters.
4. **Healing timeline** — the `H` table, then `buildHealth()` builds the DOM once and keeps mutable refs in `healthRows`.
5. **Confetti** — `buildConfetti()` mounts 14 spans once.
6. **Occasions** — `occViews()` computes, `renderOccasions()` re-renders that list (the only DOM rebuilt after boot).
7. **Money**, **install hint**, **static labels** — handlers and one-time text.
8. **`sinceParts()` / `setUnit()`** — calendar-true time split and label writing.
9. **`tick()`** — the once-a-second heartbeat, then `setInterval(tick, 1000)`.

## Conventions to follow

**Vanilla, ES5-flavored JavaScript.** `var`, `function`, classic `for` loops,
IIFEs, `element.onclick`. No `const`/`let`, arrow functions, template literals,
classes, or modules anywhere in the file. Match that style — it is deliberate,
for maximum compatibility on an older iOS Safari.

**No dependencies, no network.** Fonts and icons are embedded as data URIs
specifically so the app renders true offline forever. Never add a CDN link, a
`fetch`, an analytics snippet, or an npm package.

**Styling is two-tier.** Design tokens and reusable primitives live in the
`<style>` block as CSS custom properties and classes; per-element layout lives in
inline `style="…"` attributes on the markup and in `el.style.cssText` in JS.
Follow the existing split rather than introducing a new pattern.

**Numerals must be lining and tabular.** The old heading face defaulted to
old-style figures, and a `9`'s descender ran through the label beneath it. That
face is gone, but the belt-and-braces rule stays: every element that
shows a number sets `font-feature-settings:'lnum','tnum'`, and `body` sets
`font-variant-numeric:lining-nums`. Any new numeric element needs the same.

**Singular/plural is handled explicitly.** `setUnit()` writes "1 Week" not
"1 Weeks"; day/days strings branch on the value. Keep that up.

**Built for a 65-year-old at arm's length.** Type is large, tap targets are
generous, and copy is plain and warm — not clinical, not preachy. Text inputs
are ≥16px so iOS does not zoom on focus. Shrinking type or tightening spacing to
fit something new is the wrong trade; find another way.

**Render efficiency.** `tick()` runs every second, so it guards work behind
change checks (`lastDays`, `lastCeleb`, `lastOccKey`, `row.done`). New per-tick
work should be similarly guarded rather than rebuilding DOM each second.

## Invariants worth knowing

- **`localStorage` keys are frozen**: `jj_rate`, `jj_occasions`, `jj_hint_done`.
  Renaming one silently discards data the user has already saved. Keep reads and
  writes wrapped in `try`/`catch` (private mode can throw).
- **The icon exists in three places** and all three must stay byte-identical:
  `icon.png`, and two base64 copies at `index.html` lines 13–14 (`apple-touch-icon`
  and `icon`). A past commit shipped a re-themed app with a stale gold icon
  because only one was updated. Verify with:
  `sed -n '13p' index.html | sed 's/.*base64,//; s/">$//' | base64 -d | md5sum` — it
  must match `md5sum icon.png`.
- **`QUIT` is the single source of truth for day zero** (`new Date(2026, 6, 15)`,
  local midnight; the month is 0-indexed). The "Sober since …" label is derived
  from it at boot — but the markup also carries a matching static fallback in
  `#sinceLabel`, which is what shows in a non-executing preview. Change both
  together. Remember the direction: moving `QUIT` **earlier** raises the day
  count.
- **Hidden timeline entries are intentional.** Rows in `H` with a trailing `true`
  are withheld for later reveal; flip that entry's flag (or `SHOW_EXTENDED`) to
  show them. Don't delete them as dead data.
- **Occasions store month + day only** (`{name, m, d}`), so they recur annually;
  the "nth sober one" counts occurrences between `QUIT` and the next occurrence.
- **DST**: days-away uses `Math.round`, not `Math.ceil`, so a one-hour clock shift
  cannot add a phantom day.
- **`#jsWarn`** is visible in markup and hidden by the last line of the script. It
  is what the user sees when viewing a non-executing preview, so it must stay the
  final statement of the IIFE.
- **Only Lora is embedded.** Headings use the `--font-heading` stack, which starts
  at Palatino — a system face on iOS, so it costs nothing to ship. Cormorant
  Garamond was embedded through the Evergreen re-theme after nothing referenced
  it anymore, and was removed (~101 KB). If a heading ever needs a face iOS does
  not have, it has to be embedded as base64 like Lora — never linked.

## Previewing changes

No toolchain. Open `index.html` directly, or serve the root so the relative
`icon.png` resolves:

```bash
python3 -m http.server 8000   # then open http://localhost:8000/
```

Check at 375px and 320px widths — the counter row squeezes six units onto one
line, and label sizes were tuned against those worst cases. There are no tests
and no linter; verification is visual plus reasoning about the arithmetic.

## Git workflow

- Feature work happens on `claude/*` branches; `main` is what gets served.
- Push with `git push -u origin <branch>`.
- Commit messages follow the existing style: a short lowercase-ish summary line,
  then a prose body explaining **why** the change was made and what tradeoff it
  bought — including concrete measurements when the change is dimensional
  ("keeps 3.5px slack at 375px"). End with the `Co-Authored-By:` trailer.
- Do not open a pull request unless explicitly asked.
