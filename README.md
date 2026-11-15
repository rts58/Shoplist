# Shoplist 🧾

A small, **local-first** shopping list app that lives in your phone's browser.

Shoplist is a [Progressive Web App (PWA)](https://developer.mozilla.org/en-US/docs/Web/Progressive_web_apps) — a single `index.html` file with no backend, no accounts, and no build step. It looks and feels like a native app, works offline, and keeps all of your data on your own device in the browser's `localStorage`.

This app was built as an initial exercise in using a local LLM to create a usable phone app. All of the coding was done with LM Studio using Cline as an agent with the model Qwen3.8-27b.

```
Shopping list — Whole Foods:
- Spinach (Produce)
- Milk (Dairy)
- Eggs (Dairy)
```

---

## ✨ Features

- **Organized by store and category** — keep separate lists per shop (Grocery, Pharmacy, …), each with its own categories (Produce, Dairy, …).
- **Two views**
  - **Items** — your stores, categories, and items. Checking an item *selects* it for your list.
  - **My List** — the items you've selected, split into **To buy** and **Purchased**. Checking here marks something bought.
- **Text the list via SMS** — one tap opens your carrier's messaging app with your to-buy items pre-filled (or copy to clipboard).
- **Search** — filter items within a store.
- **Full CRUD** — add / rename / delete stores, categories, and items, with bottom-sheet dialogs.
- **Works offline** — a service worker caches the app, so it loads and runs with no connection.
- **Installs like an app** — use your browser's *Add to Home Screen* to get a full-screen, icon-bearing app.
- **100% local** — no server, no tracking, no sign-up. Your data never leaves your phone.
- **Backup & restore** — export everything to a JSON file and restore it anywhere (see [Backups](#-backups)).

---

## 📱 Getting started

### Use the hosted app

1. Open the app URL in your phone's browser (e.g. `https://rts58.github.io/Shoplist/`).
2. Tap your browser's **Share / Menu** button, then choose **Add to Home Screen**.
3. Launch it from the home-screen icon — it runs full-screen and works offline.

> ⚠️ **Your list is tied to the browser on that device.** Data is stored in that browser's `localStorage` under the key `shoplist.v1`. Clearing browsing data, or using a different browser/device, starts you with a fresh list. There is no automatic sync or cloud backup — but you can [export a backup](#-backups) and restore it elsewhere.

### Run it locally

There's nothing to install or build — it's a static site:

```bash
# from the repo root
python3 -m http.server 8000
# then open http://localhost:8000
```

Any static file server works (`npx serve`, `php -S localhost:8000`, etc.).

---

## 💾 Backups

Because everything lives in one browser, your list is at the mercy of that browser — **clearing browsing data or switching devices wipes it.** There's no sync, so a backup is your safety net.

From **Stores → ⋮ → Manage stores**, use the **Backup** row:

- **Export** — downloads your entire list (every store, category, and item) as `shoplist-backup.json`.
- **Import** — picks a backup file and restores it.

> ⚠️ **Import replaces everything.** Restoring a backup overwrites your *current* list in full — stores you added since the backup was made are gone. Shoplist has no merge. Make a fresh export before importing an older backup if you want to keep both.

The backup file is a small, self-describing JSON document you can keep anywhere (email, cloud drive, an USB stick, a git repo):

```json
{
  "app": "shoplist",
  "version": 1,
  "data": { "stores": [ "…your full list…" ] }
}
```

Import is forgiving: it accepts either the envelope above or a raw state blob, validates it, and asks you to confirm before touching your list.

---

## 🏗️ How it's built

Shoplist is intentionally a **zero-dependency, single-file** app. The entire UI and logic live in `index.html` (markup + CSS + vanilla JavaScript). There are no npm packages, no framework, and no compiler.

| File | Purpose |
|---|---|
| `index.html` | The whole app — HTML, CSS, and JavaScript in one file. |
| `manifest.json` | Web app manifest — name, theme colors, icon, and standalone display. |
| `sw.js` | Service worker — caches the app for offline use. |
| `icons/icon.svg` | App icon (vector). |
| `LICENSE` | License terms (GPL-3.0-or-later). |

**State model.** Everything is one plain object, persisted to `localStorage`:

```js
{
  stores: [
    {
      id: "...",
      name: "Whole Foods",
      categories: [
        { id: "...", name: "Produce", items: [ { id: "...", name: "Spinach", done: true, purchased: false } ] }
      ]
    }
  ],
  activeStore: 0,   // currently selected store
  openCats: {},     // which category cards are expanded
  tab: "list"       // "list" | "stores"
}
```

**The one checkbox, two jobs.** Each item has a single `done` flag that means different things per view:

- In **Items**, `done` = *selected for my list*.
- In **My List**, a second `purchased` flag moves the item from *To buy* to *Purchased*.

The service worker uses a network-first strategy for the page (so you always get the latest version when online) and falls back to the cache when offline, plus cache-first for static assets.

---

## 🚀 Deploying on GitHub Pages

Because the app is a static site, the easiest way to share it is **GitHub Pages** — free, always-on HTTPS hosting.

1. Push the repo to GitHub (see [Development](#-development)).
2. In the repo, go to **Settings → Pages**.
3. Set **Source** to *Deploy from a branch* → **`main`** branch, root **`/`** → **Save**.
4. Your app is live at `https://rts58.github.io/Shoplist/`.
5. Send that URL to friends; they *Add to Home Screen* and they're set.

> The repository name must match the Pages URL (here: `Shoplist`). Pages serves the branch root, so no extra path configuration is needed.

---

## 🔧 Development

Since there's no build step, developing is just editing `index.html` and refreshing.

```bash
# serve locally
python3 -m http.server 8000
```

**Tip:** if the app looks stale after an edit, the service worker may be serving a cached copy. Bump the cache version at the top of `sw.js` (`const CACHE = "shoplist-v8";` → `v9`) to force a refresh, or unregister it from devtools. On a phone that already has the app, the service worker re-fetches only after this version is bumped — so always bump it when you change `index.html`, `manifest.json`, or `icons/`.

### Project structure

```
Shoplist/
├── index.html        # the app (all-in-one)
├── manifest.json     # PWA manifest
├── sw.js             # service worker (offline cache)
├── icons/
│   ├── icon.svg      # app icon (browsers)
│   ├── icon-180.png  # home-screen icon (iOS apple-touch-icon)
│   └── icon-512.png  # manifest / maskable icon
├── LICENSE           # GPL-3.0-or-later
└── README.md
```

---

## 📄 License

Shoplist is free software, licensed under the **GNU General Public License version 3 or later** (GPL-3.0-or-later). See [LICENSE](LICENSE) for the full text.

In short: you're free to use, study, modify, and share Shoplist — including for commercial purposes — as long as any distributed versions remain open under the same license.

---

## 🙏 Credits

Built as a single-file PWA with vanilla HTML, CSS, and JavaScript. No frameworks, no dependencies.
