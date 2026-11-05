# 1. Architecture & Technology Stack

*Current as of November 5, 2026. Sections marked **Planned** describe the design intent for
things not built yet.*

## A. How it runs in the shop

```
   Gina's laptop  (the server today)
   ┌──────────────────────────────────────────────────────────┐
   │  start-windows.bat  →  one Node.js process                │
   │     • serves the web screens + JSON API on port 8080      │
   │     • renders the print documents                         │
   │     • takes a database snapshot every hour                │
   │                                                           │
   │  data/shop.db         ← SQLite, WAL mode: the real data   │
   │  data/uploads/        ← the shop logo                     │
   │  backups/             ← hourly snapshots (VACUUM INTO)    │
   │      └─ optional: Settings → Backup points this at a      │
   │         Google Drive for Desktop folder for off-site copy │
   └──────────────────────────────────────────────────────────┘
            ▲                    ▲                    ▲
        same laptop's        other laptop          phone / iPad
        browser              on the shop Wi-Fi     on the shop Wi-Fi

   4×6 thermal label printer  ← every document prints from the browser
```

**One server, many browsers.** Nothing is installed on the devices that use the app — they open
a bookmark. This is the most important decision for a shop with non-technical staff:

- No version drift. There is exactly one copy of the software and one copy of the data.
- A device failing on opening weekend costs nothing. Any device with a browser works.
- Two people entering orders at once cannot create two conflicting databases.
- The printer, the backups and the ticket-number uniqueness check live in one place.

**Internet independence.** Order entry, lookup, printing, pricing and status are all
local-network only. The internet is used for two non-critical things: pulling updates from
GitHub, and (once it is set up) Google Drive syncing the backups folder.

### What's different from the original plan

The original design put the server on a dedicated, always-on ~$200 mini PC running as a Windows
service. **What shipped runs on Gina's laptop**, started by double-clicking `start-windows.bat`
(or `start-mac.command`), with a console window that stays open while the app is in use.

That works, and it is how the shop runs today. The trade-off is that the app is only reachable
while her laptop is on, awake and running the start script. Moving to a mini PC is still the
right step once a second person regularly needs the app while her laptop is closed or away.
Migration is copying the `data/` folder to the new machine; no data changes are needed.

### The one real risk

The server is a single point of failure. Today's mitigations:

1. **Hourly snapshots** to `backups/`, plus a snapshot before every update (doc 6).
2. **Restore from a backup** in Settings → Backup, so recovery doesn't need a terminal.
3. **Printed paperwork** — every order already exists on paper in the shop.

**Planned:** a UPS, the always-on mini PC, and Google Drive for Desktop on the laptop syncing the
backups folder off-site.

### Alternatives considered and rejected

| Option | Why not |
|--------|---------|
| **Excel or Google Sheets as the database** | This is where the project started: a shared Excel order sheet that kept getting overwritten as people entered orders. Two editors at once corrupt rows, nothing can enforce the 5-lb minimum or ticket uniqueness, and every keystroke depends on the file being intact. |
| **Desktop app installed per laptop** | Each device needs its own install and update. Sharing one SQLite file over a Windows file share is the classic way to corrupt it. Doesn't extend to iPad. |
| **Cloud-hosted web app** | A monthly cost the shop didn't want, and it stops working when the shop internet drops — exactly when the parking lot is full. |
| **Postgres/MySQL instead of SQLite** | Another service to install, back up, and eventually fail to start after a Windows update. A season of ~1,000 orders is a few megabytes. |

## B. Technology stack

| Layer | Choice | Why this, for this shop |
|-------|--------|-------------------------|
| **Runtime** | Node.js ≥ 22.5 | One runtime for the server, print rendering and backups. Runs the same on the Mac it's developed on and the Windows laptop in the shop. |
| **HTTP server** | Built-in `node:http` with a small hand-rolled router | Two laptops don't need a framework. |
| **Database** | Built-in `node:sqlite`, WAL mode | One file, no native module to compile. Backup is `VACUUM INTO`. |
| **Migrations** | 22 numbered `.sql` files in `src/server/migrations/`, run at startup | The app upgrades its own schema when it starts. Gina never runs a migration command. A migration that rebuilds a table is marked `-- migration: no-foreign-keys` so the runner switches foreign keys off around it. |
| **Frontend** | Plain HTML, CSS and JavaScript modules — one file set per screen in `public/`, no build step | The original design called for React, TypeScript and Vite. Plain modules turned out to be enough for ten screens, and removing the build step removed the last thing that could break between the Mac and the shop. |
| **Print rendering** | Server-rendered HTML + `print.css`, printed from the browser | See doc 5. |
| **Testing** | `node --test` — 18 test files, about 200 tests | Pricing, validation, donations, restore, owner area, hunter IDs and the other rules that must never regress silently. |

### Zero runtime dependencies

```
   Shop machine needs:   Node.js  +  this folder
   No npm install. No node_modules. No native compilation. No build step.
```

The things that typically break a small deployment eighteen months later — a native module that
won't rebuild against a new Node version, a registry outage during an emergency reinstall, an
unpublished transitive dependency — cannot happen here, because none of it is present.
`package.json` lists no dependencies at all; it exists for the `npm start` and `npm test`
shortcuts.

### The folder

```
deer-processing-system/      ← a git checkout; replaced on update
   src/server/               ← server, business rules, migrations
   public/                   ← the screens
   scripts/                  ← start, update, reinstall, demo data
   data/                     ← NOT in git: the live database + uploads
      shop.db
      uploads/logo.png
   backups/                  ← NOT in git: snapshots (or the folder chosen in Settings)
```

`data/` and `backups/` are git-ignored, so an update can never touch them. The logo is stored
beside the database rather than in `public/` for the same reason: the update script runs
`git reset --hard`, which would delete anything stored inside the checkout.

### Updates

1. Changes are pushed from the development Mac to GitHub.
2. On the shop laptop, Gina double-clicks `update-windows.bat` (`update-mac.command` on a Mac).
   It saves `backups/before-update-YYYY-MM-DD.db`, then runs `git fetch` and
   `git reset --hard origin/main`.
3. She restarts the app. Migrations run on start.

The header shows the running version (`v 70a5220`), and warns when the code on disk is newer than
the code that's running — the usual reason a new feature "isn't there" after an update. A
read-only GitHub token scoped to this one repository means Gina never needs a GitHub account.
`scripts/standalone/` holds reinstall scripts so a replacement laptop can be set up without the
developer present.

### Dates and times

- Timestamps (`created_at`, `paid_at`, backups) are stored in UTC.
- `drop_off_date` is stored as the shop's **local** date. "Today" everywhere (home tiles, the
  Today quick view, the future-date check) is computed from local time. An early bug compared it
  with UTC, which put evening drop-offs on the wrong day.

## C. Concurrency: two devices, one order

**Optimistic concurrency on orders.** Every order row has a `version` integer. An edit sends the
version it loaded. If someone else saved in between, the save is refused with *"Someone else
changed this order while you had it open. Reload and try again."* Last-write-wins silently is not
acceptable when the losing write might be a cut instruction.

**Status updates are exempt.** Marking "butcher complete" is a targeted single-field write, not a
whole-order save, so it never triggers a conflict. Status is the high-frequency, low-risk action
and has to stay one tap.

## D. Security

There are no user logins. **The network boundary is the security boundary:** the app listens on
the shop network only, with no port forwarding.

Two passwords exist, and neither is network security:

- **Owner password** — keeps the season's money totals out of sight of staff (doc 3, screen 10).
  Stored hashed, set on first visit.
- **Reset password** — turns "Clear test data" in the Danger Zone from an accidental click into
  a deliberate act. It ships as a known default and is meant to be changed.
