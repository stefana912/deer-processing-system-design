# 6. Backup, Recovery & Deployment

*Current as of November 5, 2026. Sections marked **Planned** are not built yet.*

## Backup architecture

```
   ┌──────────────────────────────────────────────────────────────┐
   │  The live database — data/shop.db  (SQLite, WAL mode)        │
   │  NEVER inside a synced folder                                │
   └────────────────────────┬─────────────────────────────────────┘
                            │  when the app starts, then every hour
                            │  VACUUM INTO — a consistent copy taken
                            │  while the app keeps serving orders
                            ▼
   ┌──────────────────────────────────────────────────────────────┐
   │  Snapshots — backups/shop-YYYYMMDD-HHMM.db  (UTC time)       │
   │  or the folder chosen in Settings → Backup                   │
   │  Kept: newest 48 · then one a day for 30 days ·              │
   │        then one a month for a year                           │
   │  Plus backups/before-update-YYYY-MM-DD.db from every update  │
   └────────────────────────┬─────────────────────────────────────┘
                            │  Planned: Google Drive for Desktop
                            │  syncing that folder
                            ▼
   ┌──────────────────────────────────────────────────────────────┐
   │  Off-site copy — Google Drive                                │
   │  Covers fire, theft, drive failure, ransomware               │
   └──────────────────────────────────────────────────────────────┘
```

### The one thing that must not be done

**Never put the live `data/shop.db` inside a Google Drive, Dropbox or OneDrive folder.** Sync
clients copy files mid-write, and with SQLite's WAL files that reliably corrupts a database. Only
the backups folder, which holds finished, closed snapshots, goes into Drive.

### Why `VACUUM INTO`

It's SQLite's own consistent-snapshot command: a complete, valid database file, taken while the
app keeps serving orders, with no risk of catching half a transaction. A file copy of a live
SQLite database guarantees neither.

### Seeing that it works

Every run is recorded in `backup_log`. The Home status bar shows the last backup time, or *⚠ check
backups* if the last one failed. **Settings → Backup** has:

- **Backup folder** — where snapshots go. Blank means the app's own `backups/` folder. The setting
  checks that the folder exists and can be written to.
- **Back up now.**
- **Restore from a backup** (below).
- **Recent backups** — the last runs, with time, size and result.

### Off-site copy — Planned, one manual step

The app side is done: the backup folder can point at any folder, including one synced by Google
Drive for Desktop. What's left is installing Drive for Desktop on Gina's laptop and pointing the
setting at its folder. **This is the gap that most risks real data loss**, and it's the first item
on the to-do list (DESIGN.md).

The design chose the desktop sync client over the Drive API: no OAuth flow, no token expiry in
February, offline handled natively, and Gina can see the files herself.

## Recovery

### Restore from a backup (built)

Settings → Backup → **Restore from a backup**:

1. Choose a backup file. The app checks it really is one of its backups and shows what's in it —
   nothing changes yet.
2. Confirm with the reset password. The backup is brought up to the current schema (a backup from before an update is
   older), then its rows are copied into the live tables in one transaction: all or nothing.
3. Three things stay with **this** machine instead of coming from the backup: the backup folder
   setting, the reset password and the owner password.

A backup made by a newer version of the app is refused with *"Update this app first, then restore
it."* A file damaged by passing through something that treats it as text (a paste, or an email
that "cleans" attachments) is detected and refused.

### The laptop is dead

1. On any laptop: install Node.js and git, and run the reinstall script from
   `scripts/standalone/` (see INSTALL.md in the private repo).
2. Start the app, then restore the newest backup from Drive (or a USB copy) through Settings.
3. Point the other devices at the new laptop's address.

Until the off-site copy is set up, this only works if a backup file survived somewhere else.

### Planned

- **Automatic restore testing** — monthly, open the newest snapshot read-only, compare row counts
  with live, and show a red mark on Home if they disagree.
- **Season-end archive** — the full snapshot plus CSVs of orders, customers and sausage, readable in
  20 years without this software.
- **A weekly USB or second-machine copy**, against a Google account problem.

The original design's **Google Sheets read-only mirror** — so orders could be looked up on a phone
if the shop computer died — isn't built. It's still worth adding purely for phone lookup.

## Deployment

### What's running today

| Item | Today | Original recommendation |
|---|---|---|
| Server | Gina's Windows laptop | A ~$200 always-on mini PC, wired to the router |
| Start | Double-click `scripts/start-windows.bat`; the console window stays open while the app runs and the browser opens itself | Windows service via NSSM, starting at boot |
| Printer | One Nelko 4×6 thermal label printer, Bluetooth | 4" label printer + plain-paper laser |
| Backup | Hourly snapshots on the laptop | The same, plus Drive sync |
| UPS | None | 600 VA for the mini PC, router and printer |

The app listens on port 8080 on every network interface of the laptop, so other devices on the
shop Wi-Fi reach it at `http://<laptop address>:8080`. It is never exposed to the internet — no
port forwarding — which is why it needs no logins.

### Updates

`update-windows.bat` (or `update-mac.command`) saves a `before-update` snapshot, runs `git fetch`
and `git reset --hard origin/main`, and tells Gina to restart the app. `data/` and `backups/` are
git-ignored, so an update can't touch them. The header warns if the code on disk is newer than the
running program.

Getting git working on the Windows laptop was its own small project. PowerShell didn't recognize
`git` until it was on the PATH, and stray `desktop.ini` files that Windows drops into folders broke
the update script until it learned to clear them.

### Network checklist

1. **Reserve a fixed IP** for the laptop in the router, and write it on a card by the computer.
2. **Bookmark it** on every device that uses the app.
3. **Check for Wi-Fi client isolation.** A guest network blocks device-to-device traffic and
   silently breaks everything; shop devices must be on the main network.
4. **Firewall:** allow port 8080 on the local network only.

### Day-one checklist

Done, per the project history: the app is installed and live on the laptop, the active season is
set, and all four documents are sized for the shop's label printer. Still open, or not recorded:

- [ ] Google Drive for Desktop installed and Settings → Backup pointed at it
- [ ] Prices in Settings checked against the shop's printed price list
- [ ] A test order entered, printed, status-changed and paid end to end on the shop laptop
- [ ] A restore rehearsed once by the person who'll have to do it
- [ ] Fixed IP reserved and written down
- [ ] A one-page card by the computer: address, restart steps, restore steps, who to call

## Phones and iPads

One responsive web app, no separate install. The screens use large touch targets, numeric
keyboards on phone, ticket and pound fields, and the device's own date picker. The order screen
works well on a phone, so someone in the cut room can look up a ticket and update its status
without walking to the laptop. Order entry works on an iPad but is laid out for a laptop.
