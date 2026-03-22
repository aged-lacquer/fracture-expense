# fracture-expense
# Fracture Weekly P&L Generator

**Live URL:** `https://aged-lacquer.github.io/fracture-pl`

A standalone HTML expense categorization and P&L summary tool for Fracture's weekly financial reporting. No server, no AI, no login — runs entirely in the browser.

---

## What This Tool Does

Takes raw CSV exports from 5 bank/card accounts, lets a VA categorize each transaction as OPEX (with a category) or a non-OPEX check-route (COGS, 3PL, Shippo, Meta, etc.), and outputs a weekly P&L expense summary matching the structure of the `New Weekly P&L` tab in Google Sheets.

Vendor rules persist in the browser — once a vendor is categorized, it auto-matches every future occurrence.

---

## Architecture

```
Single HTML file (no build, no dependencies, no server)
├── CSV Parser (5 source-specific formats)
├── Vendor Rules Engine (localStorage, fuzzy prefix matching)
├── Categorization UI (keyboard-driven flashcard flow)
├── P&L Summary Generator (matches sheet OPEX structure)
└── Copy-to-clipboard output (tab-separated, paste into Sheets)
```

**Tech:** Vanilla JS, no frameworks, no external API calls. All data stays in the VA's browser.

**Hosting:** GitHub Pages (free, static file serving). Update by editing `index.html` in the repo.

---

## The 5 Expense Sources

| Source | CSV Filename Pattern | Amount Sign | Skip Rule |
|---|---|---|---|
| **Chase Checking 1815** | `Chase1815_Activity_YYYYMMDD.CSV` | Negative = expense | Skip positive amounts (revenue) |
| **DK AMEX** | `activity.csv` or `activity (N).csv` | Positive = expense | Skip negative amounts (payments) and "ONLINE PAYMENT" rows |
| **DL ChaseInk 9219** | `Chase9219_ActivityXXX.CSV` | Negative = expense | Skip "Payment" type rows |
| **DK ChaseInk 6206** | `Chase6206_ActivityXXX.CSV` | Negative = expense | Skip "Payment" type rows |
| **JL ChaseInk 7819** | `Chase7819_ActivityXXX.CSV` | Negative = expense | Skip "Payment" type rows |

**Auto-detection:** The parser identifies sources by filename (1815, 9219, 6206, 7819) or by CSV header format (AMEX has `Date,Receipt,Description,Amount`). ChaseInk cards also detected by the card number in the first column of data rows.

**CSV column layouts:**
- **Chase 1815:** `Details, Posting Date, Description, Amount, Type, Balance, Check#`
- **DK AMEX:** `Date, Receipt, Description, Amount`
- **ChaseInk (all 3):** `Card, Transaction Date, Post Date, Description, Category, Type, Amount, Memo`

---

## Transaction Routing

Every transaction gets one of three routings:

### 1. OPEX (category required)
Transaction is an operating expense. VA picks from 20 categories:

```
Creative Location, Creative Staff, Creative Equipment,
Shipping Fees, Samples, Import Fees,
Rent, Utilities, Office Supplies, Software/Subscriptions,
Salaries, Transportation, Food,
Financial Services, Interest, Legal,
Advertisement, Influencer Gifting
```

### 2. OPEX + Specifier
Same as OPEX, but also flags a specifier check for tracking. Auto-assigns to `Software/Subscriptions`. Specifiers:

```
VPM, SHOPIFY, ADOBE, GSUITE, KLAVIYO, OPENAI, SKOOL
```

### 3. Non-OPEX Check Route
Transaction is NOT operating expense — routed to a separate P&L line. Options:

```
COGS, 3PL, SHIPPO, META, TIKTOK, TAX, PAYOUT, LOAN
```

**How this maps to the Google Sheet:** OPEX items go into the `Less: OPEX (Inputs)` section (rows 37-57 of `New Weekly P&L`). Check-routed items map to their respective P&L lines (COGS → row 19, Local Shipping/SHIPPO → row 21, 3PL → row 22, Meta → row 26, TikTok → row 27, etc.).

---

## Vendor Rules Engine

### How matching works (in priority order):
1. **Exact match** — normalized vendor string matches a rule exactly
2. **Progressive prefix** — tries shorter and shorter prefixes (min 5 chars)
3. **Rule-is-prefix** — checks if any stored rule is a prefix of the vendor (e.g., rule `FACEBK` matches `FACEBK *RSEBWJHCQ2 MENLO PARK`)
4. **Contains match** — checks if any stored rule (min 6 chars) appears within the vendor string

### Normalization
Before matching, vendor descriptions are normalized:
- Uppercased
- Dates stripped (`01/15/2026` → removed)
- `#reference` numbers stripped
- `ORIG CO NAME:...` suffixes stripped (Chase checking long ACH descriptions)
- `JPMxxxxx` Zelle IDs stripped
- 10+ digit numbers stripped
- Truncated to 60 chars

### Pre-seeded rules
91 rules extracted from historical spreadsheet data (`fracture-vendor-rules.json`). Import once per browser via "Import Rules" button.

### Known limitation: New Zelle recipients
Zelle payments to the same person match (e.g., "Zelle payment to Rocky" → Creative Staff). But a brand new person the system has never seen won't auto-match because different Zelle recipients map to different categories (some are Creative Staff, some are Salaries, some are Rent, some are Office Supplies). The VA must manually categorize new Zelle recipients once.

---

## Weekly Workflow (VA Instructions)

### First-Time Setup (once)
1. Open the app URL in Chrome
2. Click **Import Rules** in the left sidebar
3. Select the `fracture-vendor-rules.json` file
4. Confirm "Imported 91 rules" toast appears

### Every Monday
1. **Set the date range FIRST** — change the date pickers at top to last Monday → last Sunday (e.g., `2026-03-16` to `2026-03-22`). The app defaults to last Mon-Sun but verify it's correct before importing.

2. **Import CSVs** — click Import CSV, then drag-and-drop (or Choose Files) all 5 CSV exports:
   - Chase1815 from Chase checking
   - activity.csv from AMEX
   - Chase9219 from DL's card
   - Chase6206 from DK's card
   - Chase7819 from JL's card

3. **Categorize** — click "Categorize" in sidebar. For each transaction:
   - If teal **auto-match badge** appears → press **Enter** to confirm
   - If no auto-match → click the correct OPEX category, or click a check-route button
   - Press **S** to skip, **Z** to undo, **/** to search categories

4. **Review** — click "P&L Summary" to see the expense breakdown. Click "Transaction List" tab to verify individual categorizations.

5. **Export** — click "Copy Summary" to copy the P&L expense totals (tab-separated), paste into the weekly column in Google Sheets. Click "Copy Transactions" for the detailed list.

6. **Clear** — click "Clear Session" when done. Rules are kept.

---

## ⚠️ Critical Reminders

### SET DATE RANGE BEFORE IMPORTING
The app filters transactions by the week date range. If dates are wrong, transactions won't appear in categorize or summary views. Always set dates first.

### AMEX "ONLINE PAYMENT - THANK YOU"
These are card payment rows (not expenses). The app auto-skips them. If you see `$9,579.03 ONLINE PAYMENT - THANK YOU` appearing, that's a bug.

### Chase Checking Positives
Positive amounts in Chase 1815 are revenue (Shopify payouts, etc.) and are auto-skipped. Revenue is tracked separately via Shopify reports, not through this tool.

### Vendor Rules Are Per-Browser
Rules save to localStorage, which is specific to one browser on one computer. If the VA switches browsers or clears browser data, she'll need to re-import the rules JSON. Recommendation: periodically click "Export Rules" to save a backup JSON file.

### This Tool Only Handles Expenses
Revenue (Product Sales, Returns, Shipping Rev, Tax Rev), COGS from Shopify, Zelle revenue, ad spend KPIs, and all CM1/CM2/CM3 calculations are still entered manually in the Google Sheet. This tool generates the OPEX section and check-route totals only.

### "Transporation" Typo
One historical rule from the spreadsheet has "Transporation" (missing a 't') as the category for `UBER *TRIP`. This came from the original sheet data. Either fix the rule or add "Transporation" as an accepted variant.

---

## P&L Output Structure

The "Copy Summary" output produces tab-separated rows in this order:

```
MARKETING OPEX
  Creative Location        [amount]
  Creative Staff           [amount]
  Creative Equipment       [amount]
  Marketing Opex Total     [subtotal]

FULFILLMENT OPEX
  Shipping Fees            [amount]
  Fulfillment Opex Total   [subtotal]

INFRASTRUCTURE OPEX
  Rent                     [amount]
  Utilities                [amount]
  Office Supplies          [amount]
  Software/Subscriptions   [amount]
  Infra Opex Total         [subtotal]

OPERATIONS OPEX
  Salaries                 [amount]
  Transportation           [amount]
  Food                     [amount]
  Operations Opex Total    [subtotal]

FINANCIAL SERVICES OPEX
  Financial Services       [amount]
  Interest                 [amount]
  Legal                    [amount]
  Financial Services Total [subtotal]

OTHER OPEX
  Advertisement            [amount]
  Influencer Gifting       [amount]
  Samples                  [amount]
  Import Fees              [amount]
  TOTAL OPEX               [grand total]

NON-OPEX CHECK ROUTES
  COGS                     [amount]
  3PL                      [amount]
  SHIPPO                   [amount]
  META                     [amount]
  TIKTOK                   [amount]
  TAX                      [amount]
  PAYOUT                   [amount]
  LOAN                     [amount]
```

All expense amounts are negative in the output (matching the sheet convention).

---

## Keyboard Shortcuts (Categorize View)

| Key | Action |
|---|---|
| `1-9, 0` | Select OPEX category 1-10 |
| `A-H` | Select OPEX category 11-18 |
| `Enter` | Confirm auto-matched rule |
| `S` | Skip transaction |
| `Z` | Undo last action |
| `/` | Focus search/filter box |

---

## Updating the App

1. Go to `github.com/aged-lacquer/fracture-pl`
2. Click `index.html` → pencil icon (edit)
3. Select all → delete → paste new code
4. Commit changes
5. Site updates in ~30 seconds

---

## Future Enhancements (Not Built)

- **AI auto-categorization:** Add Anthropic API call to suggest categories for unknown vendors. Would handle the ~20% of transactions that don't match rules.
- **Revenue inputs:** Add fields for Shopify revenue, COGS, Zelle, ad spend to generate the complete P&L column (not just expenses).
- **Full P&L output:** Compute CM1, CM2, CM3, MER, Net Profit — generate a complete column paste for the sheet.
- **CSV date pre-filtering:** Auto-filter CSVs to the selected week on import so only relevant transactions load.
- **Multi-week memory:** Track which weeks have been completed to prevent double-entry.
- **Rule conflict detection:** Alert when the same vendor gets categorized differently across sessions.
