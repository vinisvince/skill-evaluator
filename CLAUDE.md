# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

A single-file, client-side web app (`skill-evaluator.html`) that scores a
`SKILL.md` file (a Claude Code skill definition) against a fixed set of quality
criteria. The user imports/pastes skill content; the app runs regex-based
heuristics to auto-score each criterion 0–5, lets the user manually override via
sliders, then renders a radar chart, verdict, weaknesses, and prescriptive
recommendations. It is UserAdgents (UA) branded.

## Running / building / testing

There is **no build, package manager, lint, or test setup**. The entire app is
one HTML file with inline CSS and JS. To run it, open `skill-evaluator.html` in
a browser (or serve the directory, e.g. `python3 -m http.server`). External
runtime dependencies are loaded from CDNs: **Chart.js** (cdnjs) for the radar
chart and the **Inter** font (Google Fonts) — so a network connection is needed
for full rendering.

## Architecture

Everything lives in `skill-evaluator.html`. Two data structures drive the whole
app; almost all meaningful changes happen by editing them:

- **`CRITERIA`** (array of ~9 objects) — the source of truth. Each entry has
  `id`, `title`, `desc`, `low`/`high` slider labels, and an `analyze(text)`
  function. `analyze` is a pure function: it runs regex tests over the raw skill
  text and returns `{ score: 0–5, signals: [{label, type}] }`, where `type` is
  `'found' | 'warn' | 'missing'` (rendered as ✓ / ⚠ / ✗). To add or change a
  criterion, edit this array and add a matching key in `RECOMMENDATIONS`.
- **`RECOMMENDATIONS`** — keyed by criterion `id`, then by score `0`–`5`, each
  giving `{ title, body }`. The recommendation text is in **French** (the rest
  of the UI is English); keep that convention when editing. `renderRecommendations`
  looks up `RECOMMENDATIONS[id][score]`, so every criterion id needs all six
  score levels.

Runtime flow: `init()` builds the criterion cards and the chart from `CRITERIA`,
then wires file-drop / paste handlers. `analyze()` computes stats, runs every
criterion's `analyze`, and stores results in the module-level state objects
`scores`, `autoScores`, `notes`, plus `skillContent` and `chart`. `manualScore()`
overwrites `scores[id]` from a slider and flips the card's "Auto" tag to
"Manual". `updateResults()` is the single recompute path — it sums `scores`,
sets the verdict, weaknesses, and chart, and calls `renderRecommendations()`.
`exportReport()` serializes the current scores + signals to Markdown and copies
to clipboard.

## Known inconsistencies to be aware of

The criterion count and score denominators are out of sync across the file and
are easy to break further:
- The header text and the global-score denominator say **/ 45** (9 criteria × 5),
  and the verdict thresholds (`>= 36` green, `>= 23` yellow) assume a 45-point max.
- But the per-card label is hardcoded `Criterion ${i+1} / 7` and `exportReport()`
  emits `/35`. These are stale and assume a different criterion count.

If you add/remove a criterion, update all of: the `/ 45` denominator, the
`/ 7` card label, the verdict thresholds in `updateResults()`, the `/35` strings
in `exportReport()`, and the header copy ("9 criteria · 45 pts max").
