# Inventory Manager — Multi-Pass Merge (Design Spec)

**Date:** 2026-06-17
**Status:** Approved design — ready for implementation plan
**Target version:** v0.6.0 (minor — new subsystem)
**Scope:** `index.html` only (single-file, zero-backend, no new dependencies)

---

## Problem

A single inventory check is often split across multiple people and multiple passes:
the original CSV goes out, person A completes ~75% and exports, that file goes to person
B to finish, and meanwhile the GM may also be correcting added assets in Inventory Manager
off yet another copy. Every export is a divergent branch with no merge step, so:

- It's easy to mix up which file is current.
- Parallel work off different copies forces manual three-way merges in the GM's head.
- Manually-added assets appear as brand-new rows with no shared key, so the same physical
  item gets double-counted across passes.

**Root cause:** no single source of truth and no merge. The fix is to make Inventory
Manager the **reconciliation hub** — load all partial exports for one check, merge them
into one corrected dataset, and export that as the source of truth.

## Goals

1. Loading 2+ exports of the **same tab** reconciles them into one dataset instead of the
   last file overwriting the others (today's behavior).
2. The common case (different people checked different items) auto-resolves silently.
3. Genuine disagreements are **flagged, never silently overwritten**.
4. Multiply-added assets are de-duplicated when safe (matching Asset ID) and surfaced for
   review when not.
5. The reconciled result exports as a single corrected CSV (reusing v0.5.0 export).

## Non-goals (explicitly out of scope)

- No backend / live sync / accounts (stays zero-backend static HTML).
- No added-asset dedupe beyond matching Asset ID (no fuzzy description matching).
- No per-pass/per-row timestamps (Inventory Check does not emit them; "most-recent-wins"
  is therefore not available and not used).
- No change to Inventory Check (the counting app).

---

## The flow

1. Each counter works in Inventory Check and exports their partial CSV.
2. The GM loads **all** partials for one check into Inventory Manager at once (the file
   input already accepts multiple files).
3. Same-tab files **reconcile** (see rules below). Conflicts and keyless added assets are
   flagged in the exceptions panel.
4. The GM resolves flags using the existing v0.5.0 item editor.
5. **Export Corrected CSV** → the single source of truth, fed back into Inventory Check.

Merge only engages when 2+ files feed the **same tab** (assets↔assets, parts↔parts,
bulk↔bulk). An assets file plus a bulk file populate different tabs with no interaction, as
today. (Recall parts and bulk are split from one bulk-shape source file by MFR prefix; each
bulk-shape partial splits then merges into its respective tab.)

---

## Row identity (keys)

Reconciliation matches rows across files by a stable key per tab:

- **Assets:** `Asset ID` (from `detectAssetsCols().id`), trimmed; compared case-insensitively.
- **Parts / Bulk:** `PART # + "␟" + Bin Location` (`cols.id` + `cols.bin`). The same part
  number can occur in multiple bins, so part-number alone is not unique.

A row whose key is blank/missing is treated as **keyless** (see Added assets).

### Same-check guard

Before merging an incoming file into a tab that already has rows, sanity-check that they're
the same check. Warn (and let the user confirm/cancel) if **any** of:

- The detected column set differs materially (different headers for the key/status columns).
- The incoming file shares **< 10%** of its keys with the already-loaded set (likely a
  different yard/check).

The branch (`Market`) mismatch is surfaced in the warning text but is not by itself blocking.

---

## Reconcile rule (per key, across all passes)

For each key, collect every source row's decoded `_status` (`""` / `ok` / `bad` / `warn`)
and `_note`:

- **All blank** → merged status `""` (unchecked).
- **Exactly one distinct non-blank status** → that status wins (**checked beats blank**);
  the merged note is the note from the source that supplied the winning status.
- **Two or more distinct non-blank statuses** → **Conflict**. The merged row is flagged
  `_conflict` and provisionally held at the **most-severe** status (severity `bad` > `warn`
  > `ok`) so it stays visible as a problem until resolved. Identical statuses across passes
  are agreement, not a conflict.

Non-status fields (description, make/model, qty columns, etc.) are taken from the first
source row for that key; they are assumed stable across passes (they come from the same
original asset list).

### Provenance

Each merged row records `_sources`: a map of `sourceFileName → { status, note }`. This
powers the conflict view ("*Lot-A.csv: Missing · bob-export.csv: Accounted*") and builds
trust in the merged result.

### Resolving a conflict

A conflict row appears in a new **Conflicts** section. Tapping it opens the **existing
v0.5.0 editor** pre-loaded with the item; the editor additionally lists each source's
answer as quick-pick buttons. Saving a status clears `_conflict` and marks the row edited
(it becomes an authoritative value, exported normally). Resolving a conflict is just an edit.

---

## Added assets (manually-added rows)

Added rows are identified by the existing `_added` flag (`decodeAdded` of the "Added
Manually" column).

- **Added rows that share an Asset ID** (with each other, or with an existing keyed row)
  **auto-merge** into one row by the normal key path, then follow the reconcile rule above
  (agree → merged; differ → conflict).
- **Keyless added rows** (no Asset ID typed) cannot be matched. They are pooled into a new
  **"Needs review — added"** section, grouped by source file, where the GM keeps, merges, or
  deletes them. (Delete = a new lightweight per-row action; merge = manual, by editing one
  and deleting the other.)

---

## UX

All additions live in the existing exceptions panel and session strip — no new screens.

- **Two new exception sections**, rendered above the existing status sections:
  - **Conflicts (N)** — one row per unresolved conflict, showing each source's status; tap to
    resolve via the editor. Expanded by default.
  - **Needs review — added (N)** — keyless added rows grouped by source file; each row gets a
    **Delete** affordance in the editor. Expanded by default.
- **Session strip** gains: "**Merged N files**" and an unresolved-flag count
  ("**3 to resolve**") when > 0.
- **Stats / completeness / breakdown / normal exceptions / export** all operate on the
  merged dataset and update live (unresolved conflicts count at their provisional severity,
  so they surface as discrepancies until resolved).
- **Print** is unaffected (controls already stripped; flag sections print expanded like other
  exception sections).

---

## Implementation sketch

- **`applyImport` → `mergeImport(kind, headers, rows, fileName, cols)`:** if the tab is empty,
  behave as today (first import). If it already has rows, reconcile incoming rows into the
  existing set by key:
  - Build/extend a `Map<key, mergedRow>`. For an existing key, fold the incoming source's
    status/note into `_sources` and recompute merged `_status`/`_note`/`_conflict` via the
    reconcile rule. For a new key, add the row (carrying its `_sources` entry).
  - Keyless rows are appended to a `tab.keylessAdded[]` list (not the keyed map).
  - Track `tab.mergedFiles[]` (filenames) for the session strip.
- **Per-row fields added at ingest** (extend `tagEditable`): `_key`, `_sources`,
  `_conflict` (bool). Existing `_uid` / `_origStatus` / `_origNote` / `_edited` unchanged.
- **`buildExceptions`** gains `conflict` and `keyless` buckets; `renderExceptions` renders the
  two new sections (reusing `renderSection` / `renderRow`); conflict rows show `_sources`.
- **Editor (`openEdit`)**: when `_conflict`, show the per-source quick-pick buttons; on save,
  clear `_conflict`. Add a **Delete** button for keyless-added rows (removes the row from
  `tab.keylessAdded` + re-render).
- **Same-check guard** runs in `ingestFile` before `mergeImport` when the target tab is
  non-empty.
- **Export** is unchanged in mechanism (v0.5.0) — it now serializes the merged dataset
  (keyed rows + retained keyless added rows). Provisional/unresolved conflicts export at
  their current status; resolving first is recommended (the "N to resolve" count nudges this).

The merge logic (key derivation, status reconcile, severity ordering, same-check overlap %)
is pure and should be factored into small functions so it's easy to reason about, even though
the tool has no formal test harness (validate via `node --check` on the script block +
manual multi-file load, consistent with prior versions).

---

## Versioning & docs

- Bump `APP_VERSION` to **0.6.0**; update README ("What it does" + a short "Multi-pass
  merge" note; amend "Why no persistence" to mention reconcile-on-load).
- Tag `v0.6.0`; GitHub Pages (main/root) auto-deploys (~1–2 min build lag).
