# Incentive Automation API

> **Replaced a full day of monthly Excel work with a REST API + dashboard. One click now does what used to take all day.**

A production web API and manager dashboard that fully automates monthly salesman voucher-incentive calculations across 25+ outlet stores. 63,000+ voucher records, complex per-store client ownership logic, charm pricing, and Supabase/PostgreSQL backend — all replacing a manual Excel workflow that ran once a month and took hours.

Status: live in production on Cloudflare Workers (migrated from Netlify Functions — see below).

---

## The problem it replaced

Every month, someone manually:
1. Exported 60,000+ voucher rows from the POS system
2. Pasted them into an Excel workbook
3. Applied lookup formulas to match vouchers to salespeople
4. Calculated incentives by tier (new client, repeat, self-sale)
5. Applied the "owner-active" rule — does the original salesperson still work at that store this month?
6. Produced a final payout sheet per store

This took the better part of a day. With seasonal rosters and frequent store transfers, it was also error-prone — especially the ownership rule.

Now: pull from POS → click Compute → download payout sheet.

---

## How it works

```
POS System (live API)
        ↓
Cloudflare Worker pulls the full month in ONE request (~4.4s)
        ↓
POST /api/incentive-import → Supabase/PostgreSQL
        ↓
GET /api/incentive-compute → applies rates + ownership rules
        ↓
Manager dashboard — per-store breakdown, payout totals, HO report
        ↓
Export to Excel (Payout Sheet, matches the original manual workbook)
```

---

## Why it moved off Netlify — hitting a platform ceiling

The original build ran on Netlify Functions, which has a hard 10-second execution limit. The POS API returns 23.7 MB / 63,000 rows for a full month — well over that ceiling in one request. The workaround was chunking: the browser fetched 5-day slices in parallel, merged them client-side, then sent one combined import call. It worked, but every extra moving part (retry logic per chunk, partial-failure handling, merge-order edge cases) was overhead that existed only because of the platform's timeout, not the actual problem.

**Porting the same logic to Cloudflare Workers removed the ceiling entirely.** Workers don't share Netlify's short CPU-time limit for this workload, so the full month now pulls in a single request in about 4.4 seconds — faster than the old chunked approach *and* structurally simpler, since there's no chunking, merging, or partial-failure path left to maintain. The lesson: when a workaround exists purely to route around a platform limit, it's worth checking whether a different platform makes the workaround unnecessary rather than making the workaround more robust.

The migration also became the natural point to extend scope:
- **Non-voucher incentives** — birthday bonus, feedback bonus, and Google-review bonus, pulled from their own Google Sheets and folded into the same payout run instead of being tracked separately
- **Staff name normalisation** — a remap table for known aliasing (e.g. the same person appearing as "Payal" in one system and "Payel" in another)
- **Per-store editable rates** — rate overrides at the store level, not just the global rate table
- **HO consolidated report** — a head-office rollup across all stores in one view
- **Payout Sheet export** — Excel output matching the exact format of the owner's original manual workbook, so the switch to the automated system required zero change to how payouts get reviewed downstream

Verification: the new feedback-bonus numbers reconcile **11/11 exact** against the manual figures, and the HO-level total closes to the rupee once known date-typos in the source sheet are accounted for.

---

## The dashboard is now a full payout suite, not just a compute button

What used to be a single Pull → Compute screen is now a multi-tab manager dashboard:

| Tab | Purpose |
|---|---|
| **Dashboard** | Pull from POS, Compute, top-line KPIs |
| **Config** | Per-store rate overrides — edit without a code deploy |
| **F&B Points** | A manual-entry table for team-based F&B/kitchen incentive points — for staff whose contribution isn't captured by voucher sales data at all |
| **HO Report** | Head-office rollup across every store in one view |
| **Final Merge** | Combines the automated voucher payout with F&B points and the other bonus types into one number per person |
| **Staff Map** | Name-alias remap (e.g. the same person logged as "Payal" in one system, "Payel" in another) |

The point of the **F&B Points** tab specifically: not every incentive-worthy contribution comes from a POS transaction. Kitchen/F&B team performance is judged manually by a manager, so rather than force it through the automated pipeline or leave it out of the system entirely, it gets its own simple entry table that **Final Merge** then folds into the same payout sheet as everything else. One export, one number per person, regardless of whether the underlying incentive was computed or manually judged.

---

## The owner-active rule (the hard part)

The core business rule:

> A salesperson "owns" a client if they originally sold them their first voucher. But they only *receive* the incentive if they made a sale at the **same store** this month. Transfer to another store = inactive at the old store = credit goes to whoever is selling there now.

Getting this right required:
- Per-store activity check (not just "is this person still employed?")
- LEFT-flag precedence (if marked LEFT, always inactive regardless of sales)
- Owner-inactive → credit rolls to current store's active seller

The rule sounds simple. The edge cases are not: name spelling variations across POS records, people working two stores in one month, vouchers sold by people who left mid-month.

---

## Incentive rate structure

28 rate entries in Supabase covering:

| Type | Description |
|---|---|
| New client | First-ever voucher sale, by salesperson |
| Repeat client | Existing client, same salesperson |
| Self-sale | Walk-in, no assigned salesperson |
| Charm pricing | ₹5,999 / ₹7,999 (not ₹6,000 / ₹8,000) tier alignment |

Rates are stored in the database and editable — no code changes needed for monthly rate updates.

---

## Data quality lessons

Working with 63,000+ real POS rows taught some hard lessons:

- **Text dates:** POS exports date fields as `"01-Jul-26"` strings, not ISO dates — must parse manually
- **Name normalisation:** `"Staff A "` (trailing space) and `"Staff A"` are the same person — trim everything
- **Empty rows:** POS export includes subtotal and blank rows — detect by checking for missing required fields, not just blank cells
- **Charm prices:** ₹5,999 should be treated as ₹6,000 tier for incentive calculation — applied at import stage

---

## From payout to performance management

Once incentives were computed reliably, the same monthly data could answer a different question: **who is slipping, before it shows up in the store's numbers?** That became a monthly store and salesperson performance report, delivered as a formal PDF for management.

**At-risk tiers, defined so they can't be argued with:**

| Tier | Rule |
|---|---|
| Tier 1 | Sold 5+ last month, and this month is half or less (zero included) |
| Tier 2 | Sold in both months, but 3 or fewer this month |
| Watch | Down 30–49% month on month |

Grouped **store by store**, so each store manager sees their own people in one block.

**A companion conversion report** answers the question raw counts can't: is a store's salesperson performance high because it has more visitors, or because they convert better? It reports new-client rate, voucher conversion, and first-time-buyer conversion per store and per salesperson.

The report is explicit about its own limits instead of hiding them. Footfall isn't recorded per person, so an individual's conversion is measured against the *store's* footfall and doesn't adjust for days worked; that's stated in the definitions, not left for a manager to discover.

**Data traps handled:**
- In the source sheet, the first line of each store block is the **store total**, formatted exactly like a salesperson row. Read naively, every store gains a phantom top performer.
- Name variants are **flagged for review, never auto-merged**. In a report that names people as at-risk, a wrong merge is worse than a visible duplicate.
- Totals were reconciled exactly against the month's incentive export before anything was published.

---

## Key metrics

- **63,000+** voucher rows processed per month
- **18-22 stores** covered, depending on the month's active roster
- **692 vouchers** in the July run = ₹84,900 in incentives calculated
- Full-month POS pull: **4.4 seconds**, single request (down from ~14s across chunked requests)
- Non-voucher bonuses (birthday / feedback / review) now computed in the same run — feedback bonus reconciles **11/11 exact** against manual figures

---

## Tech stack

`JavaScript` · `Cloudflare Workers` · `Supabase / PostgreSQL` · `REST API` · `Google Sheets API` · `HTML/CSS`

*(Originally built on Node.js/Netlify Functions — migrated to Cloudflare Workers; see "Why it moved off Netlify" above.)*
