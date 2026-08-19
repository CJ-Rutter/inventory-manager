# Inventory Manager

GM-facing overview tool for [Inventory Check](https://github.com/CJ-Rutter/inventory-check) exports. Drop in the CSVs from a yard check, see the scoreboard, drill into the worst categories, scan exceptions.

**Version:** v0.7.5
**Created by:** CJ Rutter
**Audience:** General Managers / District Managers

## What it does

- Loads one to three CSVs exported from Inventory Check:
  - **Assets** — keyed by Asset ID + Equipment Class
  - **Parts** — bulk-shape CSV; rows whose MFR doesn't start with "BULK"
  - **Bulk** — bulk-shape CSV; rows whose MFR starts with "BULK"
- Auto-routes each CSV to the correct tab based on column shape + MFR prefix.
- Per active tab:
  - **Stat tiles**: Total / Accounted (Match) / Missing (Short) / Damaged (Over), each with a percentage.
  - **Completeness bar**: how much of the tab was actually checked.
  - **Breakdown table**: grouped by Equipment Class (assets), MFR (parts), or Bin Location (bulk). Qty-weighted for parts/bulk so 90-of-100 missing reads heavier than 1-of-1. **Discrepancy-focused** — groups that are fully accounted for (nothing missing/damaged/short/over and everything counted) are hidden by default; a "*N fully-accounted groups hidden · Show all*" toggle reveals them.
  - **Exceptions panel**: status-grouped (Missing/Damaged or Short/Over), plus a **Not Counted / Not Checked** section listing every row that was never counted (empty Check Status) — so you can see *which* items were missed, not just the completeness percentage. Sub-clustered by group key so similar problems sit next to each other. Click a breakdown row to filter exceptions to that group.
- **Multi-pass merge**: load several partial exports of the *same* check at once and Inventory Manager reconciles them into one — **including exports from different sources that spell their columns differently** (an older CSV's `Check Status` and a newer `.xlsx`'s `Status` are the same field), and **including exports that share no items at all**, which is what two people splitting a yard produce — a checked item beats a blank one; for **assets** a sighting by anyone wins, because "Missing" means *"I didn't find it in my pass"* rather than *"it isn't here"* — two people splitting a yard each mark the other's aisles Missing, and holding the row at the most-severe status used to collapse the accounted total to the items **both** found; for **parts/bulk** Match/Short/Over are all real counts, so a disagreement still holds at the most severe. That reading is the default but not an assumption you are stuck with: once a second file is loaded, an **If passes disagree** control appears — **Found wins** (default) or **Missing wins** for a shrinkage-first audit. Switching re-settles every row instantly, no re-upload, and hand-made decisions stay pinned through the switch. Genuine disagreements surface in a **Conflicts** section to resolve, and manually-added items are de-duplicated by Asset ID (keyless adds go to a **Needs review** section). Export the reconciled result as the single source of truth.
- **Fix-in-place editing** (survives later merges — a decision you make by hand outranks every file, so loading another export never quietly puts a resolved item back on the missing list): tap any item in the exceptions panel to set its **status** (Accounted/Missing/Damaged for assets, Match/Short/Over for parts/bulk) and add a **note** — as if you were re-running Inventory Check. Edits resolve **in real time**: stat tiles, completeness, breakdown, and exceptions all update instantly, and a fixed group drops off the discrepancy view. Edited rows carry an **✏ edited** tag.
- **Export corrected CSV**: write your edits back out — the active tab's original file with the **Check Status** / **Check Note** cells updated for edited rows — for archiving or to re-import once Inventory Check adds `.xlsx` import. (Counted-qty/variance are untouched; recount is out of scope.)
- **Print**: hits the browser print dialog with a layout that strips controls — GMs can forward the printed page or save as PDF.
- **Branch name** editable in the header strip; auto-pulled from the assets `Market` column when present.

## How to use

1. Open `index.html` in any modern browser. No server needed.
2. Tap **Load CSV** and pick one or more files exported from Inventory Check (Assets as `.xlsx`; Parts/Bulk as `.csv` with `Check Status` / `Counted Qty` / `Variance` columns).
3. Review the scoreboard, scan the breakdown for hot-spot categories, click a row to drill into its exceptions.
4. **Fix issues in place**: tap an exception item, set its status / add a note, Save — it resolves live. When done, tap **Export Corrected** to save the corrections.
5. Tap **Print** to forward the report.

## Why no persistence

Still no *on-device* persistence: GMs see different yards' CSVs constantly and stickiness would be more confusing than helpful — every page load starts fresh. Loading multiple files now *reconciles on load* (merges same-key rows) rather than the last file silently winning. Fix-in-place edits and merge results live only in the current session; to keep them, **Export Corrected** and re-load that file (or re-import it once Inventory Check adds `.xlsx` import — IC currently imports CSV only). Different stance than Inventory Check, which auto-saves an in-progress check.

## Versioning

Bump `APP_VERSION` in `index.html` and tag the release (`git tag vX.Y.Z`) when shipping changes. The version chip in the header and the feedback email subject both read from this constant.

## Status

v0.1.0 — skeleton. Functional bones: load, route, stats, breakdown, exceptions, print. Future work to consider:
- Multi-check history (compare today's check to last month's)
- Branch comparison view (load Marysville + Columbus, see side-by-side)
- Exception CSV export
- $ at risk tile if Inventory Check exports gain a cost/OEC column
