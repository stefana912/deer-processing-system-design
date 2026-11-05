# Deer Processing Shop — Order Management System

**Design document set — current as of November 5, 2026 (version 70a5220, migration 022)**

Built for La Famiglia Butcher Shop's processing season: about 1,000 deer a season,
September–February, with one or two non-technical people entering orders on laptops and phones
over the shop's Wi-Fi.

## The one-sentence description

A single Node.js program running on Gina's laptop, serving a touch-friendly web app to any
laptop, phone or iPad on the shop Wi-Fi, storing everything in one SQLite file, printing work
tickets and receipts on the shop's 4×6 thermal label printer, and snapshotting the database every
hour.

## Documents

| # | Document | Covers |
|---|----------|--------|
| 1 | [Architecture & Technology Stack](docs/01-architecture-and-stack.md) | How it runs in the shop, what it's built with, updates, security |
| 2 | [Database Schema](docs/02-database-schema.md) | Every table, money handling, what's snapshotted |
| 3 | [Screens & Order-Entry Flow](docs/03-screens-and-flows.md) | All ten screens, with screenshots |
| 4 | [Business Rules, Status Model & Pricing](docs/04-rules-status-pricing.md) | Every validation rule, status workflow and price |
| 5 | [Printing Architecture & Document Layouts](docs/05-printing.md) | The five printouts, the label printer |
| 6 | [Backup, Recovery & Deployment](docs/06-backup-and-deployment.md) | Snapshots, restore, updates, what's still manual |
| 7 | [Edge Cases, Roadmap & Design Decisions](docs/07-edge-cases-and-roadmap.md) | What each gap became, known gaps, the to-do list, the review answers |
| 8 | [Donation Tracking](docs/08-donation-reimbursement.md) | $5/lb, the package count, the state form |

Screenshots in `docs/images/` use made-up demo data from `scripts/seed-demo.js`.

## Status

**Live.** Installed on Gina's laptop since August 18 and the system of record for new orders.
About 210 automated tests pass. The application code and its install guide live in a private
repository; this public repository holds the design documents and the change history.

These documents began as the original design, written and reviewed before any code. They have
been updated to describe what's actually built. Where the build differs from the plan, the doc
says so and keeps the reasoning.

The shop is in New York State. No sales tax: processing is a service, not a sale of goods.

## Decisions that still hold

- **Donation earns $5/lb**, counted in 1-lb packages on the order. That count is unrecoverable
  once the meat leaves. → [doc 8](docs/08-donation-reimbursement.md)
- **Payment is collected at pickup, not drop-off.** An unpaid order is normal; meat that left
  unpaid is the exception.
- **The euro price may be blank or the standard default** ($140, plus an optional $20 plaque).
  Blank prints *TO BE CONFIRMED*. The euro mount is on the same bill with its own balance.
- **Ticket numbers repeat between years**, so the key is (season, ticket) and every printed
  document carries the season.
- **Seasons stayed simple.** A season is a label with an on/off switch.
- **The test for a new feature:** does it prevent a mistake involving a physical animal, or
  capture a number that disappears if nobody writes it down? If not, a filter on the order grid
  is probably enough. → [doc 7](docs/07-edge-cases-and-roadmap.md#the-pattern-in-these-answers)

## What's different from the original plan

| Planned | Built |
|---|---|
| A dedicated always-on mini PC running as a Windows service | Gina's laptop, started with a double-click |
| React + TypeScript + Vite + Tailwind | Plain HTML, CSS and JavaScript, no build step, zero dependencies |
| Payment at drop-off, a required PAID / NOT PAID choice | Payment at pickup, nothing asked at entry |
| Doe, buck, button buck at one flat price | Also elk, moose and specialty animals, priced per pound on the bone; All Sausage |
| Sausage and jerky | A menu of sausage, jerky, roasts, all-pork items and pack items, plus special items typed at the counter |
| No donation screen; transcribe from an export | A Donations screen that builds the state's HPNAP form |
| Call log, pickup records, batch shipments, Google Sheets mirror | Not built |
| Edits keep original prices | When prices changed, the edit asks: keep the original prices or use today's (doc 4) |
| — | Hunting companies, the customer-name fix, Names to check, the Owner area |

## What's built vs. still on the list

**Live and in daily use:** order entry and editing with the full rule set; elk, moose and
specialty animals; the sausage, jerky & other menu and special items; all four print documents
sized for the label printer; per-component status tracking; payment recording with a real date
and time; donation counts and the state donation form; customer history with repeat and copy
order; hunting companies; Hunter ID carry-forward; the name question and Names to check; the All
Orders grid with quick views and date ranges; void and restore; the Owner area with stats; a
configurable backup folder, restore from backup and a visible backup history; and a full Settings
screen.

**Not built yet** — in priority order in [doc 7](docs/07-edge-cases-and-roadmap.md#still-on-the-list):

1. Google Drive for Desktop on Gina's laptop, with Settings → Backup pointed at it — the one gap
   that risks real data loss, and a manual step, not code
2. The 24-hour pickup clock and call log (the Overdue tile reads zero until then)
3. A dedicated mini PC, once someone needs the app while Gina's laptop is closed
4. CSV / PDF export and a season-end archive

## The five decisions that matter most

1. **One server, many browsers.** Nothing is installed on the devices. If one dies mid-season,
   grab another and open the bookmark.
2. **SQLite is the transactional database.** One file, consistent hourly snapshots, no service to
   fail.
3. **Prices are copied onto the order at save time.** A December price change doesn't rewrite what
   an October customer was charged.
4. **Every printable document is a server URL.** The browser prints it today; the server can print
   the same URL silently later. That's what makes multi-printer routing a small change.
5. **Statuses are per-component, not per-order.** Butcher, sausage, cape and euro finish at
   different times, sometimes months apart. There's no such thing as "the order is done."
