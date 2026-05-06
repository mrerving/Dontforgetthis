# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

"Don't Forget This" is a zero-dependency, single-file PWA for recurring personal reminders. The entire application (HTML, CSS, JavaScript) lives in `index.html`. There is no build step, no package manager, and no framework.

## Running the App

Open `index.html` directly in a browser, or serve it with any static file server:

```bash
python3 -m http.server 8080
# then open http://localhost:8080
```

The service worker (`sw.js`) only activates when served over HTTP(S) or `localhost`, not over `file://`.

## Default Branch

`development`

## Architecture

### Single-file design
All CSS, HTML, and JavaScript are in `index.html`. The CSS uses minified shorthand and CSS custom properties (`--bg`, `--accent`, etc.) with two themes defined on `:root[data-theme="dark"]` and `:root[data-theme="light"]`. The JavaScript uses heavily abbreviated variable and function names throughout.

### Data layer (three tiers)
1. **localStorage** — always-on primary store. Keys: `reminders_v1` (array), `categories_v1` (array), `theme_v1` (string). All reads go through `ld(key, fallback)` and writes through `sv(key, value)`.
2. **OPFS (Origin Private File System)** — automatic secondary store. On every `saveAll()`, data is written to `reminders.json` in the browser's private file system via `opfsWrite()`. On startup, `initOpfs()` reads this file and restores state — OPFS data survives "Clear browsing data" unlike localStorage. Falls back silently if `navigator.storage.getDirectory` is unavailable.
3. **File System Access API** — optional user-controlled store. When the user connects a `.json` file via the Manage tab, a `FileSystemFileHandle` is persisted in IndexedDB (`dft_app` DB, `fs_handles` store) so it survives page reloads. On startup, `initFileSystem()` restores the handle and re-reads the file if permission is already granted. Writes are guarded by `_writing`/`_pendingWrite` flags to serialize rapid successive saves. FSAPI syncs are layered on top of OPFS — both run on every save.

`saveAll()` writes localStorage, triggers `opfsWrite()`, and (if a file is connected and has permission) triggers `writeFileData()`.

### Reminder data model
```js
{
  id: string,           // crypto.randomUUID()
  title: string,
  category: string,
  interval: {
    value: number,
    unit: 'minutes'|'hours'|'days'|'weeks'|'months'|'years'|'specific_dates'
  },
  specificDates?: [     // only present when unit === 'specific_dates'
    { month: number, day: number }  // up to 3 annual dates
  ],
  snooze: { value: number, unit: 'minutes'|'hours'|'days' },
  nextDue: string,      // ISO 8601
  state: 'pending'|'snoozed',
  createdAt: string     // ISO 8601
}
```

`doDone(id)` advances `nextDue`: for `specific_dates` it uses `nextSpecificDate()` to find the nearest upcoming annual date; for all other units it calls `addIv()` with the reminder's interval. `doSnooze(id)` sets `nextDue` to now + snooze duration and flips `state` to `'snoozed'`. `wakeSnoozes()` converts snoozed reminders back to `pending` when their `nextDue` passes; it runs at the top of every `renderAgenda()` call and on a 60-second `setInterval`.

### Interval types
- **Time-based** (`minutes`, `hours`, `days`, `weeks`, `months`, `years`): `addIv(date, interval)` advances a date by the given quantity and unit.
- **Specific dates** (`specific_dates`): up to 3 month/day pairs stored in `specificDates[]`. `nextSpecificDate(dates)` calculates the nearest upcoming occurrence across current and next year.

### Two views
- **Agenda** (`renderAgenda`): groups reminders into Overdue / Today / Tomorrow / Upcoming via `groupR()`. Snoozed reminders are excluded from all groups.
- **Manage** (`renderManage`): CRUD for reminders and categories, plus the file-sync status panel and export/import controls. The form toggles the specific-dates date picker panel via `showDatePanel()` when `fIU` (interval unit select) changes.

### PWA / offline
`sw.js` uses a cache-first strategy (`dft-v2` cache). On install it caches all five assets and calls `self.skipWaiting()` to activate immediately. On activate it purges stale caches and calls `self.clients.claim()` to take control of all open tabs. The cache name is the only version identifier — bump it in `sw.js` when assets change.

### Standalone download
The download button fetches the live page HTML and offers it as `dont-forget-this.html`, producing a fully self-contained offline copy with no server dependency.

## Key Abbreviations

The codebase uses short names throughout. Common ones:

| Short | Meaning |
|-------|---------|
| `R` | reminders array |
| `C` | categories array |
| `KR/KC/KT` | localStorage keys |
| `ld/sv` | localStorage get/set |
| `fmtRel` | format relative time |
| `fmtIv` | format interval display string |
| `addIv` | add interval to a date |
| `uid` | generate UUID |
| `esc` | HTML-escape a string |
| `fh` | FileSystemFileHandle |
| `FS_SUPPORTED` | File System Access API available |
| `OPFS_SUPPORTED` | OPFS available |
