# Shoplist — Native iOS App (hand-off notes)

> This document is the **bridge** between the Shoplist PWA (this repo) and the upcoming native iOS app. It defines the shared data format, the backup envelope, and the architectural decisions both apps must agree on so a user can export from the PWA and import into the native app (and vice-versa) without losing anything.
>
> If you are building the native app, read this first.

---

## 1. The two projects, one data contract

| | PWA (this repo) | Native iOS app |
|---|---|---|
| Stack | Vanilla JS, single index.html | Swift Package + SwiftUI |
| Storage | localStorage (key shoplist.v1) | SwiftData (CloudKit-syncable) |
| Offline | Service worker | N/A (native) |
| Re-uses from PWA | — | Branding, the shoplist.v1 schema, the backup JSON envelope, the two-flag item model |

**Rule:** the PWA and the native app are separate projects with separate toolchains, but they share **one canonical data format**. Anything that crosses the bridge must round-trip through that format.

---
## 2. The shoplist.v1 data schema

This is the exact shape of the app state object. The PWA stores it under the localStorage key **shoplist.v1**; the native app should adopt the same structure for its import/export layer.

### Top-level state

    {
      "stores": [ Store ],    // required; non-empty in practice
      "activeStore": 0,       // index into stores; clamped to [0, stores.length)
      "openCats": { },        // categoryId -> boolean; which category cards are expanded
      "tab": "list"           // "list" | "stores" — which bottom tab is showing
    }

### Store

    {
      "id": "starter-store-grocery",   // string, unique
      "name": "Grocery",               // display name
      "categories": [ Category ]       // manual, user-ordered sequence
    }

### Category

    {
      "id": "starter-cat-produce",      // string, unique
      "name": "Produce",
      "items": [ Item ]                 // storage order is NOT the display order (see below)
    }

### Item

    {
      "id": "starter-item-banana",      // string, unique
      "name": "Bananas",
      "done": false,                    // see "two flags" below
      "purchased": false                // optional — may be absent; treat absent as false
    }

**Field notes**

- **IDs.** The PWA generates them with crypto.randomUUID() (a timestamp+random fallback when that API is missing). Seed/starter data uses deterministic slugs (seed-*, starter-*) so the sample is stable across installs. The native app should generate UUIDs for new entities and **preserve** whatever IDs it reads from a backup (do not re-mint them on import).
- **purchased is optional.** Seed data frequently omits it. A reader **must** treat a missing purchased as false. Exporting from the native app should write both flags explicitly so the PWA always sees a complete object.
- **Display ordering.** Categories render in **stored order** (the user can move them up/down). Items within a category render **A–Z** (a sorted copy — the stored array order is untouched). If the native app wants the same UX, sort a copy for display; do not mutate storage order.

### The two flags (critical — get this right)

Each item has **one checkbox that does two different jobs** depending on the view:

| View | Flag | Meaning |
|---|---|---|
| **Items** (browsing a store) | done | *Selected for my list* |
| **My List** | purchased | *Bought* — moves the item from To buy to Purchased |

The derived list states are:

- **To buy** = done is true AND purchased is false
- **Purchased** = purchased is true
- **Not on my list** = done is false

Transitions the UI performs:

- Check in **Items** — toggle done.
- Check in **My List** — toggle purchased.
- Delete from **My List** — set done = false AND purchased = false (the item stays in the store, just unselected).
- **Clear purchased** (in the Purchased group) — for each item where purchased is true: set done = false and purchased = false. Items still in To buy are left untouched.

> The native app should mirror these exact transition rules so a list exported after a shopping trip matches what the PWA would produce.

---

## 3. The backup file (the actual bridge)

This is the file a user exports/imports. It is the **only** mechanism that carries data between the two apps.

- **Filename:** shoplist-backup.json
- **Envelope** (what exportBackup() writes):

    {
      "app": "shoplist",          // constant, identifies the app
      "version": 1,               // schema version (integer)
      "data": { /* the full state object from section 2 */ }
    }

- The PWA importer (parseBackup) **accepts both** the envelope **and** a bare state object:
  - if the top-level app field equals "shoplist" — use its data field
  - otherwise — treat the whole object as the state (a bare shoplist.v1 blob)
- **Validation** before accepting: data.stores must be an array. Then normalize:
  - activeStore — keep if it is a number in [0, stores.length), else 0
  - openCats — {} if missing
  - tab — "list" if missing
- On a bad file the PWA shows "Not a valid backup file" and aborts. The native app should do the same — validate, then either fully replace the store set or reject; never half-import.

**What the native app must do**

- **Export:** produce the full envelope { app, version, data } where data is the complete state — all stores, categories, items, with **both** done and purchased written explicitly. This file must be importable by the PWA.
- **Import:** accept the same envelope **and** a bare state object (mirror parseBackup). Preserve all IDs. Validate data.stores is an array. After import, the native app should normalize activeStore/openCats/tab just like the PWA.
- **Round-trip test:** export from PWA → import into native → export from native → import into PWA. The resulting state must be identical (modulo the optional purchased default). Keep this as a test case.

**Versioning.** The envelope version is the gate. Current: 1. If the schema ever changes in a breaking way, bump to 2 and have both apps handle both 1 and 2 (read old, write new). Non-breaking additions (a new optional field) do not require a bump.

---

## 4. Suggested SwiftData mapping

Suggested (not mandated) — the point is to keep the JSON contract stable.

    import SwiftData

    @Model final class Store {
        var id: String            // persisted JSON id
        var name: String
        @Relationship(deleteRule: .cascade, inverse: \Store.categories)
        var categories: [Category] = []
    }

    @Model final class Category {
        var id: String
        var name: String
        var sortIndex: Int        // explicit — SwiftData relationship arrays are unordered
        @Relationship(deleteRule: .cascade, inverse: \Category.items)
        var items: [Item] = []
        var store: Store?
    }

    @Model final class Item {
        var id: String
        var name: String
        var done: Bool            // selected for my list
        var purchased: Bool       // bought
        var category: Category?
    }

**Notes**

- **Explicit sortIndex.** SwiftData relationship arrays do not guarantee order, so the user-managed category order must be stored, not inferred. Items display A–Z, so they need no sortIndex.
- **Explicit String ids.** Persist the JSON id so import/export is lossless and CloudKit merge is stable. Do not rely on SwiftData persistentModelID as the wire id.
- **UI state is not synced.** activeStore, openCats, and tab are per-device UI state — keep them in @AppStorage, not in the SwiftData models. They are part of the backup file only for PWA parity; the native app can ignore or round-trip them.

### The codec

A single Codable DTO that matches the JSON exactly, plus functions to convert to/from SwiftData models. This is the one place the wire format is defined in the native app — keep it next to the PWA schema.

    struct ShoplistV1: Codable {
        var stores: [StoreDTO]
        var activeStore: Int
        var openCats: [String: Bool]
        var tab: String
    }
    struct StoreDTO: Codable { var id: String; var name: String; var categories: [CategoryDTO] }
    struct CategoryDTO: Codable { var id: String; var name: String; var items: [ItemDTO] }
    struct ItemDTO: Codable { var id: String; var name: String; var done: Bool; var purchased: Bool? }

    // Envelope for backup files
    struct ShoplistBackup: Codable { var app: String; var version: Int; var data: ShoplistV1 }

    // Round-trip: models <-> DTO
    func encode(_ stores: [Store]) -> ShoplistV1
    func decode(_ v1: ShoplistV1) -> [Store]

**Import/Export flow (native)**

- Export: collect all Store models → encode() → wrap in ShoplistBackup → write shoplist-backup.json.
- Import: read file → try ShoplistBackup, fall back to bare ShoplistV1 → validate stores non-empty → replace model graph → decode().

---

## 5. Branding to reuse

- **Name:** Shoplist
- **Artwork:** icons/icon-192.png, icons/icon-512.png, icons/apple-touch-icon.png, icons/favicon.svg (all in this repo, MIT-licensed).
- **Tagline:** "Your shopping list, everywhere."
- **Visual language** (optional but nice for parity): warm cream background, orange accent, 10px corner radius, DM Sans / DM Mono typography.

---

## 6. Source-of-truth references (in the PWA)

When in doubt, the PWA code is the reference implementation of the contract:

- Schema + seed data — seedData() in index.html
- Persistence — load() / save() in index.html (localStorage key shoplist.v1)
- Two-flag logic — storeItems() / toggleItem() / clearPurchased() in index.html
- Backup — exportBackup() / parseBackup() / importBackup() in index.html

Keep this document in sync with the PWA. If the schema changes in index.html, update section 2 and the version here.
