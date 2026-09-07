# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project

A single-file Sudoku game (`index.html`) built with [Juris.js](https://unpkg.com/juris@0.9.0/juris.js), loaded from unpkg at runtime. No build step, no package.json, no local dependencies.

Juris versions are **not semver**: `0.9.0` is newer than `0.88.2` (released
2025-08-17 vs 2025-07-26), because the minor is read as "9" and "88", not as
ordered integers. npm's `latest` tag is the only reliable signal — a `^0.88.2`
range would never resolve to `0.9.0`. Pin the exact version in the CDN URL, as
above. Note also that 0.9.0 deletes four subsystems (`DOMEnhancer`,
`HeadlessManager`, `TemplateCompiler`, `WebComponentFactory`), which is where
its 34KB raw saving comes from; this app uses none of them, so the upgrade was
a one-line change, but do not reach for those APIs.

It is a port of the sibling `sudoku-cc` repo, which implements the same game with native Web Components. The two are companion pieces: `juris_blog_post.md` and `juris_vs_webcomponents.md` argue the case for the Juris version over that one. Behaviour, API, fallback puzzle, and storage schema are deliberately identical, so a fix in one usually applies to the other — check both.

## Commands

Nothing to build, install, or test. Serve the directory:

```bash
python -m http.server 8000    # then http://localhost:8000
```

**This file is not Prettier-formatted** (unlike `sudoku-cc`). Match the surrounding hand-written style — no semicolons, two-space indent, single quotes for Juris state paths and style keys, double quotes elsewhere. Do not run `prettier --write` on it; it would reformat all ~1000 lines and bury the real diff.

No test suite. Verify by loading the page and reading the console, which logs every state transition.

**Debugging note:** `juris` is declared with a top-level `const`, so it lives in the global lexical scope and is **not** a property of `window`. `window.juris` is `undefined`; use the bare identifier `juris` in the console, or `frame.contentWindow.eval('juris')` from an iframe.

## Architecture

One `<script>` holds everything: a single `Juris` instance, plain functions for game logic and persistence, and five registered components.

### State

All state lives in one Juris store under `sudoku`, read with `juris.getState('sudoku')` and written with `juris.setState('sudoku.<key>', value)` or a partial object. The shape:

`currentPuzzle` / `originalPuzzle` / `solutionPuzzle` (9×9 `number[][]`, 0 = empty), `selectedRow` / `selectedCol` (-1 when nothing selected), `loading`, `error`, `completed`.

The same three-grid model as `sudoku-cc`: `originalPuzzle` gates editability via `canClearCell()`, `currentPuzzle` is the live board, `solutionPuzzle` drives validation and completion and may be `null` if decoding failed.

Grid updates are **immutable** — `placeNumber` and `clearCell` deep-copy `currentPuzzle`, mutate the copy, and set it back. Mutating in place will not trigger a re-render.

### Components

`SudokuGame` (branches on loading / no-puzzle / board) → `SudokuBoard` (computes geometry, lays out grid + controls) → `SudokuGrid` → `SudokuCell`, and `SudokuControls` → `ValidOptions`. Components return object descriptions, not DOM. Children given as a function re-run reactively; children given as an array do not.

### Geometry — read before touching any view

`getBoardMetrics()` is the single source of truth:

```js
const trackBudget = Math.min(window.innerWidth - 80 /* page chrome */ - 14 /* board chrome */, 453)
```

80px is the body margin plus `.container` padding, 20px per side each; 14px is the board's own 8px of inter-cell gaps plus 6px of border. `boardOuter` is the resulting outer box.

**Both the board and the loading view must size themselves to `boardOuter` and render the same controls row**, or they occupy different boxes and the layout shifts whenever the loading view appears. This is computed once per render, and there is no `resize` listener, so the grid does not re-fit until the next render.

### Persistence

`localStorage` under `sudoku-juris:game-state`. The key is namespaced because GitHub Pages puts every project on one shared `<user>.github.io` origin — an unqualified key collided with `sudoku-cc`, which would adopt and delete this app's save. `tryRestoreFromStorage()` migrates once from the old `sudoku-game-state` key, and `clearSavedGame()` removes both.

Saved shape: `current` / `original` / `solution` / `selectedRow` / `selectedCol` / `completed` / `timestamp` / `version`. Restore rejects a save that fails the shape check or is older than 7 days.

Two ordering invariants in `placeNumber()`, both of which were bugs before:

- **Save before handling completion.** Completion calls `clearSavedGame()`; saving afterwards writes the finished board straight back, so a reload restores a solved puzzle.
- **Never save an incorrect entry.** A wrong move is reverted by a 1s `setTimeout`; storage keeps the pre-placement board, which is what the revert restores. The timeout also saves, to re-sync if something else wrote during that window.

### Puzzle source

`https://sudoku-rust-api.vercel.app/api/puzzle` — an 81-char `puzzle` string and a `ref` that is base64 of the 81 solution digits (`atob`, not a solver). Any failure falls back to a hard-coded puzzle inside the catch. No timeout and no retry: a hung request leaves the loading view up indefinitely.

## Deployment

Two targets, same file — `index.html` must work unmodified on both.

- **Vercel**: https://sudoku-juris-eta.vercel.app, auto-deploying from `main`.
  Note the `-eta` suffix — plain `sudoku-juris.vercel.app` is **not** this
  project; that subdomain belongs to an unrelated Vite app, so Vercel assigned
  a suffixed alias. The `-richard-hess-projects` domains sit behind Vercel
  Authentication and return a login page to unauthenticated requests, so use
  the `-eta` alias when checking what is live.
- **GitHub Pages**: https://eswat2.github.io/sudoku-juris/, built from `main` / root; `.nojekyll` disables Jekyll

No build step on either, so the deployed bytes are exactly the repo's `index.html` — `curl -s <url> | md5` against `md5 index.html` tells you whether a target is current.

## Docs

`juris_blog_post.md` and `juris_vs_webcomponents.md` are advocacy pieces comparing this port to `sudoku-cc`. Their headline metrics were corrected in September 2026: the original "800 lines → 300 lines, 70% reduction" was never measured and was wrong (the real figures are 947 → 767 non-blank script lines, about 19%), and the framework was listed at 75KB when it is 133KB raw / 22KB gzipped (v0.9.0; v0.88.2 was 167KB / 29KB). Per-category line breakdowns and the "time to interactive" row were estimates presented as data and have been removed rather than restated. **Measure before adding any number to these files.**

## Known gaps

- **No accessibility support**: no ARIA, no keyboard navigation, no focus management. The grid is unusable without a pointing device.
- **Cells fall below the 32px touch target under ~390px** (25px at 320px, 31px at 375px) — the cost of sizing the board so it actually fits.
- The error branch (`!currentPuzzle`) renders an unstyled `div.error` with no box and no controls row, so it does not match the other views' geometry. It is close to unreachable, since the API fallback always supplies a puzzle.
- The Juris CDN tag has no `integrity` hash, and the app is broken if unpkg is unreachable.
- No tests, no CSP, no request timeout.
