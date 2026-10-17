# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Overview

A single-file, dependency-free quiz app (`index.html`) for practicing the MBZUAI master's admission test. All HTML, CSS and JS live inline in that one file; the only external resources are Google Fonts and the Firebase compat SDK (gstatic). There is no build, lint or test tooling — serve it with `python3 -m http.server` (Google sign-in doesn't work from `file://`).

## Architecture (inside `index.html`)

- **Question bank**: `TOPICS` maps one-letter codes (P, L, E, C, Y, A, D, G, O, S) to topic names. Questions are added one per line via `q(topic, text, options, answerIndex, explanation, extras)`, which pushes onto `Q` with `id` from a running counter `N` (shown as "Book #id", matching the prep book's numbering). Saved stats and missed lists key on these ids, so never reorder or delete `q(...)` lines; to drop a question, replace its line with `gap();` so later ids stay stable. Append new questions at the end.
  - Always exactly 4 options; `answerIndex` is 0-based into the unshuffled options.
  - `extras` may include `c` (code/matrix block rendered monospace) and `f` (a note about a fix to the book's original question). `f: DUP` marks a duplicate-option fix; any other `f` string is counted in the UI as a "corrected wrong answer".
- **Rendering**: no framework. A tiny `el(tag, attrs, ...kids)` helper builds DOM; screens are re-rendered wholesale with `app.replaceChildren(...)`. Main screens: `renderSetup()` (config, stats, weak-areas panel), `renderQ()` (current question), `finish()` (results).
- **Quiz state**: global `S` holds the active session (`mode` practice/exam, `items` with per-question shuffled option order `ord` and `pick`, current index, optional timer `end`). `cfg` holds setup choices.
- **Persistence**: `localStorage` via the try/catch-wrapped `store` helper. Keys: `mbz-cfg` (settings), `mbz-missed` (question ids answered wrong), `mbz-hist` (past scores), `mbz-topics` (per-topic `{r, n}` right/total counts), `mbz-session` (in-progress quiz, restored on load by `resume()`). Writes to the synced keys (`mbz-cfg`, `mbz-missed`, `mbz-hist`, `mbz-topics`) set `mbz-dirty` and trigger a debounced push.
- **Cloud sync**: the `sync` object pushes the synced keys to Firestore `mbzuaiProgress/{uid}` (its own collection, since the Firebase project is shared with other apps) after Google sign-in. The first sign-in on a device merges local and remote (missed lists unioned, past scores combined, per-topic stats keep whichever side has more answers); after that, a device's unsynced changes (`mbz-dirty`) win, otherwise the remote copy is adopted. `mbz-synced-uid` records which account the device last synced with. Firestore rules must restrict `mbzuaiProgress/{uid}` to its owner.
- In practice mode skipped questions are left out of scoring and stats; in exam mode unanswered questions count as wrong.
- **Theming**: colors are CSS custom properties on `:root`, with dark-mode overrides under both `prefers-color-scheme: dark` and `[data-theme="dark"]` — update both blocks when changing colors.
