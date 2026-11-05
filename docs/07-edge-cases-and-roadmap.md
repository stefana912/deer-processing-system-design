# 7. Edge Cases, Roadmap & Design Decisions

*Current as of November 5, 2026. This doc started as the list of gaps in the original
requirements. Each item now says what happened to it. Status key: ✅ built · 🟡 partly built ·
⬜ not built.*

## Edge cases from the original design

| # | Edge case | Status | Where it stands |
|---|---|---|---|
| 1 | Sausage and euro leave in batches, not one at a time | ⬜ | No shipments table or batch actions. Each order's status is updated on its own screen. |
| 2 | A deer can't supply more meat than it has | ✅ | WARN when sausage, jerky and burgers need more than 55 lbs of deer (`yield.warn_deer_lbs`). |
| 3 | Donation earns per pound, so the count has to be captured | ✅ | Package count on the order, $5/lb, plus the state form (doc 8). |
| 4 | Harvest tag number | ✅ | Optional on every order, prompted on donations, required to put a deer on the state form. |
| 5 | No record of where the meat physically is | 🟡 | `storage_location` exists and shows on the order screen, but there's no way to enter it yet. |
| 6 | Direct thermal fades with heat | 🟡 | Known trade-off: the one label printer prints customer copies too (doc 5). |
| 7 | Split and partial pickups | 🟡 | Each component has its own *Picked up* status, so sausage and meat can go at different times. No pickup records (who, when). |
| 8 | Abandoned meat | ⬜ | No abandoned state or report. |
| 9 | Deer arriving in unusable condition | ⬜ | Notes and void cover it today. |
| 10 | Duplicate customer records | 🟡 | No merge tool. The name question (doc 4) stops new duplicates and overwrites, and Names to check repairs past ones. |
| 11 | Family members sharing a phone | ✅ | Phone isn't unique, and a different name on a known phone asks "spelling fix or different person?" |
| 12 | Capes collected by a taxidermist | ✅ | Not applicable: the cape goes home with the meat. |
| 13 | Antler and rack identification | ✅ | A handwritten antler-description line on the euro ticket, plus the return tag. |
| 14 | Deposits and refunds | ✅ | Multiple payments per order, each with a real date and time; one payment can cover meat and euro. |
| 15 | Sales tax | ✅ | None. Processing is a service in New York State. |
| 16 | The clock on the server | ✅ | Timestamps in UTC, drop-off dates in local time. An early bug put evening drop-offs on tomorrow; fixed August 18. |
| 17 | Printer offline when saving | ✅ | Printing never blocks the save. |
| 18 | Ticket numbers repeat between years | ✅ | Unique per season, the season printed on every document, a duplicate check as the number is typed, and the number can be corrected on an edit. |
| 19 | Season rollover in January | ✅ | Kept deliberately simple: a label switched once a year. |
| 20 | Several deer from one customer at once | 🟡 | The same-day warning and NEXT ORDER (F2). No "another deer for this customer" button yet. |
| 21 | Weight-based pricing | ✅ | Built for elk, moose and specialty animals. Deer stay at one flat price. |
| 22 | Photos | ⬜ | |

## Edge cases nobody listed

The requirements and the design missed these. Each one surfaced after the shop started using the
app, and each one changed the data model, not just a screen.

| Discovered | What reality said | What changed |
|---|---|---|
| Aug 18 | "Today" was computed in UTC, so evening drop-offs landed on tomorrow | Local dates throughout |
| Aug 21 | The real printer was a 4×6 label printer | Every document resized |
| Sep 12 | NYS DEC needs a Hunter ID on every deer, but blocking the save was worse than a blank | New column, a flag instead of a block, later carried forward between seasons |
| Sep 20 | The shop processes elk, then moose, then other animals, priced by weight on the bone | `deer_type` widened three times, each a full table rebuild |
| Oct 4 | The state donation form is a real, recurring chore | A Donations screen and a printable state form — reversing "deliberately not built" |
| Oct 4 | A prepaid bill looked like a refund once the euro mount was added | Meat and euro settle as one bill with two balances |
| Oct 5 | Deer come in under hunting groups | Companies as their own list, matched regardless of spelling |
| Oct 9 | Typing a new name on a known phone was silently renaming the customer | The name question, `customer_names`, and Names to check |
| Oct 14–Nov 3 | Menu items aren't all sausage: jerky is all venison, pastrami is a whole roast, Canadian Bacon is all pork, burgers sell by the pack | What an item is made of moved onto the item |
| Oct 24 | Some customers want the whole deer as sausage | All Sausage, its own flat price |
| Nov 5 | The owner doesn't want season totals visible to staff | The Owner area behind a password |
| Nov 5 | Editing a paid order's phone number quietly re-priced it at new prices | The counter chooses: keep the original prices or use today's |

## Known gaps in what's built

- **The Overdue pickup tile always reads 0**, and the Home printer indicator always reads Ready.
- **All Orders shows at most 300 orders** and has no export.
- **`print.auto_print_on_save`** is a setting nothing reads.

## Still on the list

In priority order:

1. **Install Google Drive for Desktop on the shop laptop and point Settings → Backup at it.** The one
   gap that risks real data loss, and the only step left that isn't code.
2. **The 24-hour pickup clock and call log**, so the Overdue tile means something and storage fees
   have a factual basis.
3. **A dedicated always-on mini PC**, once someone needs the app while the owner's laptop is closed.
4. **CSV / PDF export from All Orders, and a season-end archive.**
5. A storage-location field, a customer merge tool, batch sends to processors, and a manual print
   size override — each when a real need shows up.

## Timeline of what shipped

The full list is in CHANGELOG.md. In short:

| Dates | What landed |
|---|---|
| Aug 16 | First version, built in one evening: order entry, customers and history, the order list, pricing, validation, hourly backups, tests, and the design docs — 43 files, ~6,400 lines |
| Aug 17–18 | The four printouts, status tracking, install scripts, editing after printing, Settings and logo, payment recording, reprint prompts, void and restore, configurable backup folder. **Live in the shop on day 3.** |
| Aug 21 | Resized for the real label printer; self-service reinstall |
| Sep 12–20 | Hunter ID, ticket correction, donation totals, date ranges, Stats, editable sausage menu, elk, restore from backup |
| Oct 4–9 | State donation form, version display, one-bill euro, moose and companies, Hunter ID carry-forward, the name fix |
| Oct 14–Nov 3 | Jerky, special items, All Sausage, specialty animals, roasts, Hamburgers |
| Nov 5 | Owner area; Clear test data fixed; the keep-or-update price choice on edits; "All this season" limited to the active season |

More than 60 commits and 22 database migrations in 12 weeks.

## Design decisions from the original review

The design went through two rounds of questions with the shop before any code was written.
Kept as a record of the reasoning, with what happened since.

### Round 1

| # | Question | Answer | Effect | Since then |
|---|---|---|---|---|
| 1 | Ticket tags / barcodes | Plain numbers, no barcode, ranges repeat yearly | Key is (season, ticket); season shown everywhere | Holds |
| 2 | Euro processor cost | Per head, varies | Euro price optional — quote it or leave it TBD | Holds; default $140 added |
| 3 | Donation reimbursement | Per pound, handled by hand | Capture the count; no claim workflow | The state form was built in October |
| 4 | Harvest tag | Not required by the state; needed when the shop keeps the deer | Optional, prompted on donations | Holds |
| 5 | Partial donation pricing | Full price | As designed | Holds |
| 6 | Cape pickup | Goes with the meat | Checklist item, no workflow | Holds |
| 7 | Sales tax | None | Removed from the design | Holds |
| 8 | Payment timing | At drop-off | Required PAID/NOT PAID choice | **Reversed: payment is at pickup** |
| 9 | Storage fee | Owner's discretion | No configured rate | Holds; nothing drives it yet |
| 10 | SMS | Manual for now | Text as a contact method later | Not built |
| 11 | Other services | None now, maybe later | The product catalog handles it | Elk, moose, specialty, roasts, burgers and special items all arrived |

### Round 2

| # | Question | Answer | Effect | Since then |
|---|---|---|---|---|
| A | Coordinator's donation form | Don't have it | No claim printout | **Built in October** once the form was in hand |
| B | Annual cap | Lifted | Nothing to track | Holds |
| C | What $5/lb is measured on | 1-lb packages, counted | A package count, no scale | Holds |
| D | Claim cadence | As needed | Nothing to build | Holds |
| E | Euro quoting | Quote if known, else TBD | Euro price optional | Holds |
| F | Season boundary | Not a concern | A label with an on/off switch | Holds |

### The pattern in these answers

Five of the six round-2 answers pushed toward removing something. **This shop's problem is not
insufficient software.** The parts of the system that earn their keep are the ones tied to a
physical animal in a building — ticket numbers, cut instructions, what's ready, who paid, what
the state needs.

The test for a new feature: *does it prevent a mistake involving a physical animal, or capture a
number that disappears if nobody writes it down?* If not, a filter on the order grid is probably
the right amount of software.

The weeks since launch add a second lesson. Most changes weren't new screens; they were facts
about the business that the first data model assumed away: one kind of animal, one way to price
it, one kind of menu item, one name per phone. When the shop takes on different kinds of orders or
wants different reports, expect the data model to change, not just the screens.
