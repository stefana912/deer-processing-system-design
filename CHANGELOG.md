# Changelog

Every change to the app since the first commit, newest first, from the project's git history:
60 commits from August 16 to November 5, 2026, plus 2 pull-request merges.

Each line is the commit message as written at the time. Read from the bottom up, it's a record of
what the first version didn't know about the business.

## November 5, 2026

- Fix "clear test data": it failed as soon as there was any customer
- Owner screen: title only, no description of what's behind it
- Owner area: money totals behind a password

## November 3, 2026

- Set-aside line names burgers: "for sausage, jerky & burgers"
- Canadian Bacon is not a roast
- Add Hamburgers to Jerky & Other: sold by the pack, 70% venison

## October 25, 2026

- Set the roasts aside on their own line, so nothing looks left out
- Roast is not an option: only Smoked Pastrami and Canadian Bacon are roasts
- Menu editor: drop the per-item "made of" dropdown; only Canadian Bacon is pork
- Sausage & Other / Jerky & Other: pastrami and Canadian bacon are roasts, not sausage

## October 24, 2026

- Add Specialty animals: one per-pound price, optional searchable name
- Add an All Sausage butchering option ($150, no cuts)

## October 14, 2026

- Jerky is 100% venison; add special items to the order

## October 9, 2026

- Hunter ID carries over from season to season
- Stop customer names being overwritten; companies become their own list
- Keep a hunter's Hunter ID on all their deer this season
- Carry a hunter's Hunter ID to their next deer this season

## October 5, 2026

- Add moose, and an optional company / hunting-group name on each order

## October 4, 2026

- All Orders: make the whole table reachable without scrolling the page
- Donations: never fail silently, and say when the app needs a restart
- Show a partial-payment status for the whole bill, euro mount included
- Put the euro mount on the regular bill so a prepaid bill isn't a "refund"
- Donations: remove a deer from a form and send it back to the list
- Windows update: clear stray desktop.ini from .git, and stop blaming the internet
- Show which version is running, and warn when a restart is needed
- Donations: stop relying on browser pop-ups for Add to form
- Donations report: fill in the missing Hunter ID, tag number and weight in place
- Add the state donation form: a Donations report and a printable HPNAP form

## September 20, 2026

- Mac start script: open the browser on PORT when one is set
- Add Restore from a backup to Settings > Backup
- Stats: switch between daily, weekly and monthly reports
- Elk is butchering only: no cape, euro or head choice
- Elk weight is on the bone: say so on the form, the receipt line and in Settings
- Add elk: butchered at a per-pound price, with elk wording on all paperwork
- Update the sausage list from Gina's order form, and make it editable

## September 19, 2026

- Add a Stats tab with week-by-week charts and a comparison table
- Filter All Orders by a date range, with totals for that range
- Show the donated $ on each order in a customer's history
- Rename the All Orders Total column to Order total
- Add a Donated $ column to All Orders, left of Total
- Show how many donations still need weighing in the reimbursement total
- Docs: donation reimbursement is $5/lb with no annual cap

## September 13, 2026

- Allow the ticket number to be corrected on an edit

## September 12, 2026

- Stop blocking orders on a missing Hunter ID number
- Add Hunter ID number, required by the NY DEC on every deer

## August 21, 2026

- Add a safe test mode for the reinstall scripts
- Add self-service reinstall so Gina isn't blocked on Stefan for a new laptop
- Log backlog item: manual print size override / force 2 labels
- Resize print templates for the real 4x6 label printer Gina bought

## August 18, 2026

- Add order void/restore and a configurable, verifiable backup destination
- Update DESIGN.md status — the system is live, not pending review
- Add README
- Fix "Today's drop-offs" using UTC instead of the shop's local date
- Require whole-pound donation weights; fix stale euro settings help text
- Add reprint prompts, payment recording, euro plaque, admin reset, and sausage gating

## August 17, 2026

- Update order processing
- Add Settings with logo upload; new head, donation, cut and payment rules
- Add order editing for changes after the ticket is printed
- Add print templates, status updates, and install scripts

## August 16, 2026

- Initial deer processing system
