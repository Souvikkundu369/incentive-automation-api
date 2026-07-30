# Incentive Automation API

> **Replaced a full day of monthly Excel work with a REST API + dashboard. One click now does what used to take all day.**

A production web API and manager dashboard that fully automates monthly salesman voucher-incentive calculations across 22 outlet stores. 63,000+ voucher records, complex per-store client ownership logic, charm pricing, and Supabase/PostgreSQL backend — all replacing a manual Excel workflow that ran once a month and took hours.

Live: `jusjumpin-incentive.netlify.app`

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
Browser pulls voucher data in 5-day chunks (63k rows, ~14s total)
        ↓
POST /api/incentive-import → Supabase/PostgreSQL
        ↓
GET /api/incentive-compute → applies rates + ownership rules
        ↓
Manager dashboard — per-store breakdown, payout totals
        ↓
Export to Excel
```

**Why chunked pull?** The POS API returns 23.7 MB / 63,000 rows — well over Netlify's 10-second function timeout. The browser fetches 5-day slices in parallel, merges them client-side, then sends one `/incentive-import` call to Supabase. This sidesteps the timeout entirely.

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
- **Name normalisation:** `"Ankita "` (trailing space) and `"Ankita"` are the same person — trim everything
- **Empty rows:** POS export includes subtotal and blank rows — detect by checking for missing required fields, not just blank cells
- **Charm prices:** ₹5,999 should be treated as ₹6,000 tier for incentive calculation — applied at import stage

---

## Key metrics (July 2026)

- **63,000+** voucher rows processed per month
- **18 stores** covered
- **692 vouchers** in July payout run = ₹84,900 in incentives calculated
- Processing time: ~14s (vs ~4-6 hours manually)

---

## Tech stack

`JavaScript` · `Node.js` · `Supabase / PostgreSQL` · `Netlify Functions` · `REST API` · `HTML/CSS`
