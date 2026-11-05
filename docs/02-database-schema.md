# 2. Database Schema

*Current as of November 5, 2026 (migration 022). The schema is defined by the numbered files in
`src/server/migrations/`, which are the source of truth; this document explains them.*

## Principles

1. **Money is stored in integer cents.** Never floats. `15000` = $150.00.
2. **Prices are snapshotted onto the order.** Every charge line stores the amount as calculated
   when the order was saved, and every sausage line stores its name, unit price and deer pounds.
   Changing a price in Settings does not rewrite a saved order. (Editing an order re-prices it —
   see doc 4.)
3. **Customer, order and each service are separate.** One customer → many orders. One order →
   many menu lines, many charges, one status row per service.
4. **Nothing is deleted.** Voiding sets a flag. History is permanent. The one exception is
   *Clear test data* in the Danger Zone, which exists for wiping practice orders.
5. **Components are rows, not columns.** `butcher`, `sausage`, `cape` and `euro` are values in
   `order_components`, so the dashboard is one query and a component's status lives in one place.

## Entity relationship overview

```
   seasons ──< orders >── customers ──< customer_names
                 │  │
     companies ──┘  ├── butcher_details  (1:1, butcher orders only)
                    ├──< order_items      (sausage, jerky & other menu lines)
                    ├──< order_charges    (every dollar on the receipt)
                    ├──< order_components (status per service) ──< status_history
                    ├──< payments
                    ├──< print_jobs
                    └──> donation_forms   (the state form a donated deer went on)

   products (the menu) · settings · settings_history · event_log · backup_log
```

## Tables

### `customers`

One row per person, not per deer.

| Column | Type | Notes |
|---|---|---|
| `id` | INTEGER PK | |
| `first_name`, `last_name` | TEXT NOT NULL | The person's own name. A company goes in `companies`, not here. |
| `phone` | TEXT NOT NULL | Digits only: `5705551234`. What search matches on. |
| `phone_display` | TEXT | `(570) 555-1234` |
| `phone_alt` | TEXT | Digits only, also searchable |
| `email`, `address_line`, `city`, `state`, `zip` | TEXT | Optional; not on any screen yet |
| `notes` | TEXT | Notes about the customer, not any one deer |
| `is_merged_into` | INTEGER FK → customers | For a future merge tool; excluded from search when set |
| `names_checked_at` | TEXT | Set when "Name is right" is pressed on Names to check |
| `created_at`, `updated_at` | TEXT | UTC |

Indexes: `phone`, `phone_alt`, `(last_name, first_name)`.

**Phone is not unique.** Families and hunting groups share a number. When a known phone is typed
with a different name, the order form asks whether it's a spelling fix or a different person
(doc 4) — it never guesses.

### `customer_names` *(October 2026)*

Every name a customer has had: `customer_id`, `first_name`, `last_name`, `seen_from`, `seen_to`,
`source` (`backup` | `renamed`).

Before October 2026, typing a different name on an order that matched a customer by phone
silently renamed that customer, and all their earlier deer took on the new name. The fix made it
a question. On first start after the fix, the app read every customer's name back out of the
hourly backups (`source = 'backup'`) so past overwrites show up on **Names to check**. Every rename
since is recorded as it happens (`source = 'renamed'`).

### `companies` *(October 2026)*

Hunting groups and companies: `name` as first entered, and a unique `key` (lower-case, with
spaces and `. , ' -` removed) so "MMT", "mmt" and "M.M.T." are one company.
`orders.company_id` tags each deer with the company it came in under, or none.

There is deliberately no membership table. A hunter's companies are the ones their deer came in
under. That covers a hunter in several groups and a group with many hunters, with nothing to keep
in step.

### `seasons`

| Column | Type | Notes |
|---|---|---|
| `id` | INTEGER PK | |
| `name` | TEXT UNIQUE | `"2026 Season"` |
| `is_active` | INTEGER 0/1 | Exactly one active. New orders get it. |

**A season is a label Gina switches once a year in Settings.** No start or end dates, no date
arithmetic. A deer dropped off on January 3, 2027 goes into whatever season is active — the 2026
season — which is correct. The season is stored on the order at entry.

### `orders`

One row per animal. The center of the system.

| Column | Type | Notes |
|---|---|---|
| `id` | INTEGER PK | Internal only |
| `ticket_number` | TEXT | Typed from the physical tag. TEXT so `0421` keeps its leading zero. |
| `season_id` | FK → seasons | |
| `customer_id` | FK → customers | |
| `hunter_name` | TEXT | When the person dropping off isn't the hunter |
| `company_id` | FK → companies | Optional; for reports only, never printed |
| `deer_type` | TEXT | `doe` · `buck` · `button_buck` · `elk` · `moose` · `specialty` |
| `specialty_animal` | TEXT | What a specialty animal is (pig, boar, bear…), as typed; optional, searchable |
| `animal_lbs` | REAL | Weight on the bone for elk, moose and specialty — their butchering is priced per pound |
| `harvest_tag_number` | TEXT | Optional in general; prompted on donations; needed for the state form |
| `hunter_id_number` | TEXT | The hunter's license number, which NYS DEC asks for on every deer. Not required to save. Carried forward to the hunter's next deer (doc 4). |
| `drop_off_date` | TEXT | Shop-local date; defaults to today |
| `drop_off_time` | TEXT | |
| `svc_butcher`, `svc_sausage`, `svc_cape`, `svc_euro` | INTEGER 0/1 | Cape and Euro are mutually exclusive (CHECK constraint) |
| `head_disposition` | TEXT | `took_head` · `cape` · `euro` · `shop_disposes` |
| `donation_type` | TEXT | `none` · `full` · `remaining` |
| `donated_lbs` | REAL | Count of 1-lb packages. NULL = not counted yet. |
| `donated_weighed_at` | TEXT | |
| `donation_rate_cents_per_lb` | INTEGER | Snapshotted when the count is entered (500 = $5.00/lb) |
| `donation_amount_cents` | INTEGER | `donated_lbs × rate` |
| `ammo_type` | TEXT | `NL` (non-lead) · `L` (lead), for the state form. NULL means the default, non-lead. |
| `state_form_id` | FK → donation_forms | The state donation form this deer went on |
| `state_submitted_at` | TEXT | |
| `euro_cost_cents` | INTEGER | What the outside processor charges the shop. Column only; not on a screen yet. |
| `storage_location` | TEXT | Which cooler or shelf. Shown on the order screen; no input for it yet. |
| `package_count` | INTEGER | Column only; not used yet |
| `subtotal_cents`, `adjustment_cents`, `adjustment_reason`, `total_cents` | | Calculated on save (doc 4) |
| `payment_status_meat`, `payment_status_euro` | TEXT | `unpaid` · `partial` · `paid` · `not_applicable`, derived from `payments` |
| `status` | TEXT | `active` · `void` — the record's status, not processing status |
| `void_reason`, `voided_at` | TEXT | A reason is required to void |
| `notes` | TEXT | General notes; print on the butcher order when there are no butcher notes |
| `notes_butcher`, `notes_sausage`, `notes_euro` | TEXT | Print on that document only |
| `version` | INTEGER | Optimistic concurrency — doc 1 |
| `created_at`, `updated_at` | TEXT | UTC |

**Ticket uniqueness:**

```sql
CREATE UNIQUE INDEX idx_orders_ticket_season
  ON orders(season_id, ticket_number) WHERE status != 'void';
```

Unique within a season, and a voided order releases its number. **Ticket ranges repeat every
year**, so the real key is (season, ticket), and every printed document carries the season.

**Why `deer_type` grew.** The first schema allowed three values. Elk (migration 012), moose (014)
and specialty animals (018) each needed the CHECK constraint widened, and SQLite can't change a
CHECK in place, so the whole `orders` table was rebuilt three times. Each rebuild copies every
column by name inside one migration with foreign keys switched off. That cost is the main
argument for a more general order model if the shop keeps adding kinds of work.

### `butcher_details`

1:1 with orders that have Butcher.

| Column | Values |
|---|---|
| `backstrap` | `whole` · `half` · `third` · `quarter` (NULL = none) |
| `steaks`, `roasts`, `stew`, `ground` | `basic` · `more` · `less` · `none` |
| `all_sausage` | 0/1 — the whole deer goes to sausage, no cuts (October 2026) |
| `donate_remaining` | 0/1 — keep the listed cuts, donate the rest |
| `keep_instructions` | Free text for a partial donation, e.g. "Backstrap and head only" |

CHECK constraints block More steaks + More roasts, and More stew + More ground.

### `products` — the menu

What's offered under Sausage & Other and Jerky & Other. Edited in Settings → Sausage, Jerky &
Other.

| Column | Notes |
|---|---|
| `category` | `fresh` · `smoked` · `jerky` (the Jerky & Other tab) |
| `name` | Unique per category, case-insensitive |
| `sort_order` | Button order on the order form |
| `meat` | `mix` (sausage: the Settings ratio, 60% deer) · `venison` (all deer: jerky, pastrami) · `pork` (no deer: Canadian Bacon) |
| `cut_as_roast` | 1 = one whole roast, never ground (Smoked Pastrami) |
| `pack_lbs`, `pack_pieces`, `piece_word`, `pack_price_cents` | For items sold by the pack. Hamburgers: 2.5 lb, 6 burgers, $25.00 a pack. |
| `deer_ratio` | An item's own deer share, overriding `meat` (Hamburgers: 0.70) |

Prices per pound live in Settings by category ($12 fresh, $14 smoked, $14 jerky & other), not on
each product.

### `order_items`

One row per menu line on an order. Everything is snapshotted so a reprint in March matches the
paperwork that went out in November, even if the menu changed.

| Column | Notes |
|---|---|
| `product_name_snapshot`, `category_snapshot` | As they were when saved |
| `qty` | Finished pounds |
| `unit_price_cents`, `line_total_cents` | Snapshotted |
| `deer_lbs_required` | `qty × deer share`, stored so the printed number never changes |
| `meat`, `cut_as_roast` | Copied from the product |
| `packs`, `pack_pieces`, `piece_word`, `pack_price_cents` | For pack items |
| `notes` | |

Before migration 005, sausage existed only as charge descriptions ("Sweet Italian — 10 lb").
`scripts/backfill-order-items.js` rebuilt structured lines for those early orders.

### `order_charges`

Every dollar on the receipt. The customer copy is rendered from this table, so the receipt can
never disagree with the total.

| Column | Notes |
|---|---|
| `kind` | `base_butcher` · `extra_cut` · `cape` · `euro` · `euro_plaque` · `sausage` · `jerky` · `special` · other |
| `description` | Printed on the receipt |
| `amount_cents` | Signed |
| `payable_group` | `meat` · `euro` |
| `source` | `auto` (calculated) · `manual` (special items typed at the counter) |
| `reason` | |

`payable_group` keeps the euro mount, which comes back weeks later, apart from the meat. Both
appear on one bill (doc 4).

### `order_components`

One row per service the order has: `component` (`butcher` · `sausage` · `cape` · `euro`),
`status`, `updated_at`. The states per component are in doc 4.

Also present for the planned pickup clock, but not written by anything yet: `ready_at`,
`notified_at`, `pickup_deadline_at`, `picked_up_at`.

### `status_history`

Append-only: `order_id`, `component`, `from_status`, `to_status`, `changed_at`, `actor`, `note`.
`actor` is `'shop'` until there are logins.

### `payments`

`order_id`, `applies_to` (`meat` · `euro`), `amount_cents`, `method` (`cash` · `check` · `card`
· `other`), `paid_at`, `notes`.

`paid_at` is a real date and time chosen on screen, because payments are often entered after
the fact. A payment can be removed if it was entered by mistake. Multiple rows support partial
payments. The meat and euro statuses are recalculated from these rows on every change (doc 4).

### `donation_forms` *(October 2026)*

One row per state donation form printed: `created_at`, `header_json` (business, payee and food
bank block as printed), `rows_json` (one entry per deer, the form's columns). Kept as a snapshot
so a form reprints exactly as it was sent. Orders point back via `orders.state_form_id`.

### `print_jobs`

`order_id`, `doc_type`, `printer_target`, `status`, `copy_number`, `requested_at`,
`completed_at`, `error`. One row per actual print, so the second print of a document is stamped
**REPRINT — copy 2**. Opening a document just to look at it doesn't count.

### `settings` and `settings_history`

Key/value with `value_type` (`money` · `number` · `text` · `bool`), `label` and `group_name`,
edited on the Settings screen. Every change is recorded in `settings_history` with the old and
new value.

| Group | Keys (defaults) |
|---|---|
| prices | `price.base_butcher` (15000) · `price.all_sausage` (15000) · `price.cape` (7000) · `price.euro` (14000) · `price.euro_plaque` (2000) · `price.osso_bucco`, `price.cutlets`, `price.bone_in_chops` (4000 each) · `price.elk_per_lb`, `price.moose_per_lb`, `price.specialty_per_lb` (180 each) · `rate.fresh_sausage_per_lb` (1200) · `rate.smoked_sausage_per_lb` (1400) · `rate.jerky_per_lb` (1400) |
| processing | `sausage.min_lbs_per_flavor` (5) · `sausage.deer_ratio` (0.60) · `donation.rate_cents_per_lb` (500) · `pickup.deadline_hours` (24) · `yield.warn_deer_lbs` (55) |
| shop | `shop.name` · `shop.address` · `shop.phone` · `shop.tagline` · `shop.logo` |
| donation_form | the header block of the state form: business name, payee, address, phone, processor name, food bank, food bank contact and phone |
| printers | `print.auto_print_on_save` (stored; not wired to anything yet) |
| backup | `backup.folder` (blank = the app's own `backups/`) |
| owner, danger | `owner.password` (hashed, never shown) · `admin.reset_pin` |

**No sales tax anywhere.** Processing is a service in New York State; there is no tax field,
rate or receipt line.

### `event_log`

`entity_type`, `entity_id`, `event_type`, `summary`, `payload_json`, `actor`, `created_at`.
`summary` is a plain-English sentence written at the time — *"Donation weighed — 22 lbs at
$5.00/lb = $110.00"* — so history is readable without a developer.

### `backup_log`

`kind`, `target`, `bytes`, `row_counts_json`, `status`, `started_at`, `finished_at`, `error`.
Drives the backup status on the home screen and the Recent backups list in Settings.

## In the original design, not built

The original schema also had `customer_contacts` (a call log), `pickups` (who picked up what and
when), `shipments`/`shipment_orders`/`processors` (batch sends to the sausage and euro
processors), and `orders.amount_paid_cents`. None of them exist. Doc 7 tracks which are still
wanted.
