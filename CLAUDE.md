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

### Data layer (two tiers)
1. **localStorage** — always-on primary store. Keys: `reminders_v1` (array), `categories_v1` (array), `theme_v1` (string). All reads go through `ld(key, fallback)` and writes through `sv(key, value)`.
2. **File System Access API** — optional secondary store. When the user connects a `.json` file, a `FileSystemFileHandle` is persisted in IndexedDB (`dft_app` DB, `fs_handles` store) so it survives page reloads. On startup, `initFileSystem()` restores the handle and re-reads the file if permission is already granted. Writes are guarded by `_writing`/`_pendingWrite` flags to serialize rapid successive saves.

`saveAll()` writes both localStorage and (if a file is connected and has permission) the JSON file.

### Reminder data model
```js
{
  id: string,           // crypto.randomUUID()
  title: string,
  category: string,
  interval: { value: number, unit: 'minutes'|'hours'|'days'|'weeks'|'months'|'years' },
  snooze: { value: number, unit: 'minutes'|'hours'|'days' },
  nextDue: string,      // ISO 8601
  state: 'pending'|'snoozed',
  createdAt: string     // ISO 8601
}
```

`doDone(id)` advances `nextDue` by the reminder's interval from now. `doSnooze(id)` sets `nextDue` to now + snooze duration and flips `state` to `'snoozed'`. `wakeSnoozes()` converts snoozed reminders back to `pending` when their `nextDue` passes; it runs at the top of every `renderAgenda()` call and on a 60-second `setInterval`.

### Two views
- **Agenda** (`renderAgenda`): groups reminders into Overdue / Today / Tomorrow / Upcoming via `groupR()`. Snoozed reminders are excluded from all groups.
- **Manage** (`renderManage`): CRUD for reminders and categories, plus the file-sync status panel and export/import controls.

### PWA / offline
`sw.js` uses a cache-first strategy (`dft-v1` cache). On install it caches all five assets; on activate it purges stale caches. The cache name is the only version identifier — bump it in `sw.js` when assets change.

### Standalone download
The download button fetches the live page HTML and offers it as `dont-forget-this.html`, producing a fully self-contained offline copy with no server dependency.
