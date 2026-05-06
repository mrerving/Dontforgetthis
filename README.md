# Don't Forget This

A zero-dependency, single-file Progressive Web App for managing recurring personal reminders. No accounts, no servers, no installs required — open `index.html` and you're running.

---

## Features

### Reminders & Scheduling
- **Recurring reminders** with flexible intervals: minutes, hours, days, weeks, months, or years
- **Specific annual dates** — pin a reminder to up to 3 fixed month/day dates that repeat every year (e.g., birthdays, anniversaries, annual checkups)
- **Snooze** — postpone any reminder by a custom duration (minutes, hours, or days); snoozed items reappear automatically when the snooze expires
- **Categories** — organize reminders with fully customizable, renameable, deletable categories (defaults: General, Health, Work, Personal)

### Agenda View
- Reminders grouped by urgency: **Overdue**, **Today**, **Tomorrow**, **Upcoming**
- Overdue items highlighted with a red left border for immediate visibility
- Relative time labels ("in 3h", "yesterday", "in 5 days")
- Auto-refreshes every 60 seconds; snoozed items wake up automatically

### Data Persistence (Three Layers)
1. **localStorage** — always active; data is saved instantly on every change
2. **Origin Private File System (OPFS)** — automatic background backup to the browser's private storage; survives "Clear browsing data" on supported browsers (Chrome 86+, Edge 86+)
3. **File System Access API** — optionally connect a real `.json` file on your device; the app reads from and writes to it on every save, enabling cross-device sync via cloud-synced folders (e.g., Dropbox, iCloud Drive, OneDrive)

### Data Portability
- **Export JSON** — download a full snapshot of your reminders and categories
- **Import JSON** — restore from a snapshot, with a choice to merge or replace
- **Download standalone app** — download `dont-forget-this.html`, a fully self-contained single file that runs offline with no server

### PWA / Install
- Installable as a home screen app on Android and iOS
- Full offline support via service worker (cache-first, all assets cached on install)
- Light and dark themes with toggle button; preference persisted across sessions

---

## Getting Started

### Open directly
```
open index.html
```
Basic functionality works immediately. The service worker (offline caching) requires HTTP — see below.

### Serve locally
```bash
python3 -m http.server 8080
# then open http://localhost:8080
```
Or any static file server (nginx, `npx serve`, VS Code Live Server, etc.).

### Deploy to GitHub Pages
Push `index.html`, `manifest.json`, `sw.js`, `icon-192.png`, and `icon-512.png` to a `gh-pages` branch (or configure Pages to serve from `main`). No build step needed.

---

## Files

| File | Purpose |
|------|---------|
| `index.html` | The entire application — HTML, CSS, and JavaScript |
| `sw.js` | Service worker: cache-first offline strategy, cache version `dft-v2` |
| `manifest.json` | PWA manifest: name, icons, theme color, display mode |
| `icon-192.png` | PWA icon (192×192) |
| `icon-512.png` | PWA icon (512×512) |

---

## How It Works

### Data model

Each reminder is stored as a plain JavaScript object:

```js
{
  id: "uuid",
  title: "Take medication",
  category: "Health",
  interval: { value: 1, unit: "days" },
  // for specific_dates only:
  specificDates: [{ month: 3, day: 15 }, { month: 9, day: 15 }],
  snooze: { value: 2, unit: "hours" },
  nextDue: "2025-06-01T08:00:00.000Z",
  state: "pending",   // or "snoozed"
  createdAt: "2025-01-01T00:00:00.000Z"
}
```

### Save flow

Every change calls `saveAll()`, which:
1. Writes to `localStorage` (synchronous, always)
2. Writes to OPFS (`reminders.json` in the browser's private storage) — async, fire-and-forget
3. Writes to the user-connected file via the File System Access API — async, if a file is connected and has permission

### Startup flow

1. Reads `localStorage` for immediate render
2. `initOpfs()` — reads OPFS backup and updates state if available
3. `initFileSystem()` — restores the IDB-persisted file handle; if permission is already granted, reads the file and syncs state

### Service worker

`sw.js` uses a cache-first strategy. On install it pre-caches all five assets and calls `self.skipWaiting()` to activate immediately. On activate it calls `self.clients.claim()` to take control of all open clients and purges stale caches. To force a cache refresh after updating assets, bump the `CACHE` constant in `sw.js`.

---

## Using the File Sync Feature

The File System Access API integration lets you connect a `.json` file on your device — useful for backing up data or syncing across devices via a cloud-synced folder.

1. Go to the **Manage** tab
2. Under **Data File**, click **Create new file** (to start fresh) or **Open existing file** (to import from a previous export)
3. The file handle is remembered in IndexedDB; on future visits the app reconnects automatically if permission is still granted
4. If permission lapses (common after browser restart), a yellow banner appears — click **Reconnect** to re-grant access

---

## Development

No build tools. Edit `index.html` and reload.

**Default branch:** `development`

When making significant changes to `index.html` or other cached assets, bump the `CACHE` constant in `sw.js` (e.g., `dft-v2` → `dft-v3`) so returning visitors get the updated files instead of the cached version.

See `CLAUDE.md` for detailed architecture notes, abbreviation glossary, and guidance for AI-assisted development.
