# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project overview

Single-file HTML app (`index.html`) for a World Cup 2026 prediction game (quiniela) in Spanish. No build system, no dependencies, no package manager. Open the file directly in a browser or serve it with any static file server.

## Running locally

```
# Any of these work:
python -m http.server 8080
npx serve .
# Or just open index.html directly in the browser
```

## Architecture

Everything lives in `index.html` in three sections:

1. **CSS** — inline styles, dark theme (`#0d1b2a` background), mobile-first layout capped at 600px.
2. **ES module `<script type="module">`** — initializes Firebase and exposes the Firestore API via `window._fb` so the non-module script can use it.
3. **Regular `<script>`** — all app logic (match data, render functions, Firestore reads/writes).

### Firebase / Firestore bridge

Firebase SDK is loaded from CDN. The module script exposes:
```js
window._fb = { db, collection, doc, getDoc, getDocs, setDoc, deleteDoc }
```
`waitForFirebase()` polls for `window._fb` before `init()` proceeds.

There are two Firebase configs in the file: a commented-out dev project (`sporters-dev-mundial`) and the active production project (`mundial-humberto`).

### Firestore data model

| Collection | Doc ID | Contents |
|---|---|---|
| `picks` | `<userName>` | `{ [matchId]: { home, away } }` — user's predictions |
| `users` | `<userName>` | `{ name }` — registry of all participants |
| `config` | `locks` | `{ [matchId]: boolean }` — admin-controlled locks |
| `config` | `results` | `{ [matchId]: { home, away } }` — official scores |
| `config` | `pagos` | `{ "<matchId>_<userName>": boolean }` — payment tracking |
| `config` | `mas_state` | `{ pozos: [...] }` — active prize pools for the MÁS side game |
| `mas` | `<matchId>` | settlement record for each MÁS-liquidated match |

### Match data

`GROUPS` defines 12 groups × 4 teams. `MATCH_DATES` maps `"TeamA|TeamB"` (alphabetically sorted) to date/time/venue. The `MATCHES` array is built from round-robin pairs, then knockout placeholders are appended (R16 through Final).

### Scoring

`calcPoints(pick, result)` returns:
- `0` — no result yet or no pick
- `1` — exact score match (shown as "exacto")
- `2` — pick exists but score is wrong (shown as "boleto")

The leaderboard (`renderTabla`) sorts by `bolPts` (bol × 2) descending, with exactos as tiebreaker.

### MÁS side game

Parallel prize-pool game layered on top of the quiniela. On each settled match, a new pool of `participants × 2` points is created. If any eligible participants (those who bet on that match) guessed the exact score, all active pools are split among them; otherwise pools accumulate. Settlement (`settleMas`) and reversal (`resetMas`) are admin-only.

### Admin panel

Protected by a hardcoded password (`ADMIN_PASS = "humberto26"`) stored in `sessionStorage`. Allows locking/unlocking matches, entering results, tracking payments, and liquidating MÁS pools.

### User identity

Username is stored in `localStorage` (`quiniela_user`). No authentication — anyone who knows a username can log in as them.
