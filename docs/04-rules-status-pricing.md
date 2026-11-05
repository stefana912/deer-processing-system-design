# 4. Business Rules, Status Model & Pricing

*Current as of November 5, 2026. The rules live in `src/server/lib/validation.js`,
`pricing.js` and `status.js`, with the matching prevention in `public/new-order.js`. The server
re-checks everything the browser prevents.*

## Validation

Three enforcement levels, applied deliberately:

| Level | Behavior | When it's used |
|---|---|---|
| **PREVENT** | The control is disabled or the choice is changed for you, with a one-line note. | Contradictory cut and service choices |
| **BLOCK** | Save is refused with a plain-English message that names the fix. | Missing or impossible data |
| **WARN** | Save stops once, shows the message, and goes ahead when it's confirmed. | Unusual but legitimate situations |

An employee should not be able to build an invalid order and then be told about it.

### PREVENT

| Rule | How |
|---|---|
| The head goes one place | Cape, Euro and Took Head are a single choice; picking one replaces the other. |
| More steaks + More roasts | Picking one dims the other ("Steaks and roasts come from the same cuts"). |
| More stew + More ground | Same treatment. |
| Cutlets | Cut from the steak meat: adding them sets steaks to Less and dims More and Basic. |
| Bone-in chops | Cut from the backstrap: adding them sets backstrap to None and dims the other options. |
| Full donation | Turns Butcher and Sausage off; nothing comes back to the customer. The head choice stays. |
| All Sausage | Clears the cuts and extra cuts, turns Sausage on, and cancels "donate remaining". |
| Elk, moose and specialty animals | No head choices: butchering only. |
| Sausage below the minimum | A new flavor starts at 5 lb, and the stepper won't go lower. Pack items (Hamburgers) start at one pack. |
| Butcher / sausage / euro sections | Only shown when that service is on. |

### BLOCK

| Rule | Message |
|---|---|
| Ticket number missing | "Enter the ticket number from the tag on the deer." |
| Ticket already used this season | "Ticket #4833 is already used by Sue Kowalski (dropped 11/19). Check the tag." Also shown as the number is typed. |
| First or last name missing | "Enter the customer's first and last name." |
| Phone under 10 digits | "Enter a 10-digit phone number — this is how the customer gets called for pickup." |
| No animal chosen | "Select Doe, Buck, Button Buck, Elk, or Moose." |
| Weight that isn't a number | "Enter the weight as a number of pounds, like 450." |
| Drop-off date in the future | "The drop-off date is in the future. Check the date." |
| No service and no donation | "Select at least one service, or mark this as a donation." |
| Two head choices | "Cape and Euro Mount cannot both be selected — the head can only go one place." |
| Cape or euro on elk, moose or specialty | "… can't have a cape or euro mount — it is butchering only." |
| Full donation with Butcher or Sausage | Explains the difference between a full donation and "Donate all remaining meat". |
| "Donate remaining" without Butcher | Needs Butcher, since it describes the meat left after the cuts. |
| More + More on steaks/roasts or stew/ground | Server-side guard behind the PREVENT rule. |
| All Sausage without Sausage | "All Sausage needs Sausage turned on, with the flavors the customer wants." |
| Butcher with every cut None | "This butcher order has no cuts. Select at least one, or use Donate Remaining Meat." |
| Cutlets with steaks More/Basic, chops with a backstrap | Server-side guards behind the PREVENT rules. |
| Sausage on with no flavors | "Add at least one sausage flavor, or turn off Sausage." |
| A flavor under the minimum | "Sweet Italian is 3 lbs. The minimum is 5 lbs per flavor. Increase it or remove it." |
| Special item with no description or no price | Asks for the missing half. |
| Adjustment with no reason | "Enter a reason for the price adjustment." |
| Void with no reason | Required, and stays on the record. |
| Donated count of zero, negative, a decimal, or over 90 | Counted in whole 1-lb packages; over 90 asks for a double-check. |

**Hunter ID number is not a block.** NYS DEC asks for it on every deer, so the field is labeled
*required by NY DEC* and the order screen flags it when it's missing. It shipped as a hard block on
September 12 and was relaxed the same evening: a hunter who doesn't have it on hand shouldn't be
stuck at the counter.

### WARN

| Rule | Message |
|---|---|
| Same phone, same day, another deer | "Dave Miller already dropped off #4829 today. Is this a second deer?" → *Yes, second deer* |
| Lots of meat for sausage | "This order needs about 62 lbs of deer meat for sausage and jerky. A typical deer yields 40–55 lbs boneless. Confirm with the customer." Threshold is `yield.warn_deer_lbs` (55). |
| Cape or euro on a doe or button buck | "Doe with a euro mount — is that right?" |
| Full donation with a cape or euro | The customer is still charged for the head service. |
| Donation with no harvest tag number | "This deer is being donated, so the shop keeps it. Record the hunter's tag number if they have it." |
| Elk, moose or specialty with no weight | The butchering price can't be worked out yet; the receipt says *to be confirmed* until the order is edited with a weight. |
| A company typed into the name | "The name 'MMT Graziose' includes the company 'MMT'. First and last name should be the hunter's own name." |

### The name question

When the phone matches a customer on file but the typed name doesn't, saving asks one question:

- **Fix spelling** — same person, the name on file was wrong. Renames them on all their deer, and
  records the old name in `customer_names`.
- **Different person** — someone else using this phone (a household or hunting group). This deer
  goes to that person, found or created.

### Hunter ID carry-forward

A hunter's license number doesn't change between seasons, so the app offers it again:

- New Order shows the number from the customer's most recent deer that has one.
- Saving a number fills it into that customer's other deer that have none. It never overwrites a
  different number, and skips voided orders, deer belonging to another named hunter, and deer
  already sent on a state donation form.

## Status model

There is deliberately **no overall order status.** Butcher, sausage, cape and euro finish at
different times, sometimes months apart. Each is a row in `order_components` that exists only if
the order has that service.

| Component | States, in order |
|---|---|
| **Butcher** | Not started → In process → Complete → Picked up |
| **Sausage** | Not started → Meat ready → At processor → Back in shop → Picked up |
| **Euro** | Ready to send → At processor → Back in shop → Picked up |
| **Cape** | In process → Ready → Picked up |

- One large button advances to the next state, worded as the action: *Start butchering*, *Mark
  complete*, *Send to processor*, *Returned to shop*, *Mark picked up*. A *change to…* menu sets any
  state.
- **Sausage can't pass Not started until butchering has started.** The sausage meat is cut out of
  the deer, so it can't be ready before the deer is being cut.
- Every change is written to `status_history`.
- **Ready to call** (a quick view and a Home tile) is derived: every butcher, sausage and cape
  component is complete, ready or back in shop.

### Payment

Payment is collected **at pickup**, not drop-off. The original design assumed drop-off (with a
required PAID / NOT PAID choice on every new order); Gina corrected it. So an unpaid new order is
normal, the
order form asks nothing about payment, and the exception worth flagging is meat that left the
shop without being paid for: the **Picked up, not paid** tile and **Unpaid** quick view.

The euro mount is on the same bill as the meat but has its own balance, because the skull comes
back weeks later. Statuses are derived from `payments`, never set by hand:

1. Each part (meat, euro) first takes the payments recorded against it.
2. Anything a part doesn't need spills over to the other, so one payment for the whole bill
   doesn't look like a meat overpayment and an unpaid euro.
3. Only money left over after both parts are covered is a refund.

Each part reads `unpaid`, `partial`, `paid` or `not_applicable`, and the receipt and order screen
show the overall bill as paid, partly paid or unpaid.

### Voided orders

`active` or `void`, separate from processing status. A voided order is out of every view except
**Voided**, frees its ticket number, can't be edited, and can be restored.

## Pricing

Calculated on the server on every save; the browser's total is a preview. All amounts are integer
cents. Defaults are in Settings → Prices.

```
  Butcher (deer)            $150 flat                 not charged on a full donation
  Butcher — All Sausage     $150 flat, no cuts
  Butcher (elk, moose,      $1.80/lb × weight on the bone     no line until weighed:
           specialty)                                          receipt says TO BE CONFIRMED
  Extra cuts                $40 each — Osso Bucco, Cutlets, Bone-in Chops
  Cape                      $70
  Euro mount                $140 default, editable per order, may be blank (TO BE CONFIRMED)
  Euro plaque               +$20, its own line
  Fresh sausage             $12/lb finished
  Smoked sausage            $14/lb finished
  Jerky & Other             $14/lb finished, or a pack price (Hamburgers $25 a pack)
  Special items             whatever is typed at the counter
  ───────────────────────────────────────────────────────────────
  subtotal                  sum of the lines
  adjustment                signed, requires a reason ("large deer")
  TOTAL                     subtotal + adjustment
```

Every non-zero amount is written as a row in `order_charges`, so the printed receipt is a
rendering of stored data rather than a second calculation.

### Donation pricing

| Scenario | Charged to the customer |
|---|---|
| Full donation, nothing else | **$0.00** |
| Full donation + cape or euro | The head service only |
| Partial donation (keep some cuts, donate the rest) | **Full normal price** — the shop does the same work |

Separately, every donated pound earns the shop **$5.00 from the state program** (doc 8). A partial
donation pays twice: the processing fee plus the reimbursement on the remainder.

### What deer meat each item needs

| Item | Deer share of each finished pound |
|---|---|
| Sausage (`mix`) | `sausage.deer_ratio`, 60%; the processor supplies 40% pork |
| Jerky, Smoked Pastrami (`venison`) | 100% |
| Canadian Bacon (`pork`) | 0% — all pork |
| Hamburgers | 70%, the item's own ratio |

`deer_lbs_required` is stored on each line. The butcher order prints the total as **SET ASIDE
N LBS DEER MEAT**; that's what tells the cut room how much meat to hold back before cutting.
Smoked Pastrami is a whole roast, so it prints on its own line under *ROASTS — SET ASIDE · DO NOT
GRIND* instead of being added to the grind total.

The menu taught this the hard way. Between October 25 and November 3, Canadian Bacon went from
sausage, to a roast, to "not a roast", as the actual rules came out one screen at a time.

### Price snapshotting, and edits

When an order is saved, every price used is copied into `order_charges` and `order_items`.
Changing a price in Settings doesn't touch saved orders or their reprints.

Before November 5, 2026, editing an order silently re-priced it at the current Settings prices,
so an older order edited after a price change moved to the new price.

**Editing an order after a price change asks which prices to use** (`src/server/lib/repricing.js`,
November 2026). The edit is priced two ways — at the prices the order was taken at, and at today's
— and when the totals differ, saving stops with:

> **Prices have changed since this order was taken**
> At the prices it was taken at: **$210.00** · At today's prices: **$225.00**
> Butcher Processing: $150.00 → $165.00 · The customer has already paid …
> [ Go back ] [ Use today's prices ] [ **Keep original prices** ]

- **Keep original prices** is the highlighted choice. It's for edits that have nothing to do with
  money — a phone number, a misspelled name — often on an order that's already paid. Every line the
  order already had keeps the price it was taken at. A line added in this edit (a cape that wasn't
  there before) is priced at today's price, since it had no earlier price.
- **Use today's prices** is for edits made to bring the price in line. Everything is priced from
  today's Settings and menu, as a new order would be, including sausage, jerky and pack prices.
- When the totals agree, nothing is asked.
- The order's history records which was chosen, e.g. *"kept the original prices (current prices
  would be $165.00)"*.

The original prices aren't stored as settings anywhere, but every one of them is on the order's
saved charge lines, so they are read back from there: the butcher fee, All Sausage, the per-pound
rate (charge ÷ weight), extra cuts, cape and plaque. Menu lines carry their own unit and pack
prices. While editing, the form's running total previews the original prices.

### Adjustment, not override

```
   Standard calculated:   $230.00
   Adjustment:            + $30.00   "Large deer — extra processing time"
   ─────────────────────────────────
   Total:                 $260.00
```

An adjustment keeps the standard price, the difference and the reason as three visible facts. An
override would lose the answer to "what would this normally have cost?" A negative adjustment
covers a discount.
