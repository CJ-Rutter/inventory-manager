# Inventory Manager: consume the new Assets `.xlsx` (import + round-trip export)

**Date:** 2026-07-02
**Repo:** inventory-manager (single-file `index.html`, no build, no framework)
**Target version:** v0.7.0
**Companion:** Inventory Check v1.7.0 changed the Assets export to a concise, color-coded `.xlsx` (spec: inventory-check `docs/superpowers/specs/2026-07-01-...`). This spec adapts IM to consume it and round-trip corrections back.

## Problem

Inventory Check now exports **Assets** as a real `.xlsx` (store-only ZIP, inline strings, 9 fixed columns, colored rows) instead of CSV, and defaults every asset to **Missing** until accounted for. IM currently only reads CSV, keys on the old `Check Status`/`Check Note`/`Added Manually` headers, treats blank status as "not checked," and exports corrections as CSV — none of which match the new format.

## The Assets `.xlsx` contract (from IC v1.7.0)

- Store-only (uncompressed) ZIP; `xl/worksheets/sheet1.xml` holds cells as inline strings (`<c ...><is><t>value</t></is></c>`).
- Columns, in order: `Asset ID · Cat Class · S/N · Make · Model · Market/Branch · Status · Added · Notes`.
- `Status` values: `Accounted For` / `Damaged` / `Missing` — **`Missing` is the default** (every asset has a status; no blank state).
- `Added` = `Yes` for manually-added rows, else blank.
- Row fills are cosmetic (IM ignores them); the `Status` column is the source of truth.

## Goals

1. Import the new Assets `.xlsx` into IM's existing ingest/merge pipeline.
2. Keep reading old Assets CSV **and** Parts/Bulk CSV (backward compatible).
3. Reflect Missing-by-default correctly in the scoreboard.
4. Round-trip: export corrected **Assets** back as the same colored `.xlsx`.

Non-goals: reading Excel/Sheets-**re-saved** xlsx (DEFLATE-compressed) — detected and rejected with a helpful message; changing Parts/Bulk behavior; a shared code module across the two apps (intentional duplication, per existing pattern).

---

## Part A — File loading & format dispatch

The `#file` change handler (`index.html:1300`) reads every file as text → `ingestFile(text, name)` (`index.html:581`), which calls `parseCSV` first.

Refactor:
- Extract everything in `ingestFile` **after** `parseCSV` into **`ingestGrid(grid, fileName)`** (grid = array of string-arrays, unchanged shape). `ingestGrid` does headers → `classifyCSV` → enrich → decode → `mergeImport`, exactly as today.
- `#file` handler dispatches per file by extension (case-insensitive):
  - `*.xlsx` → `FileReader.readAsArrayBuffer` → `parseXlsxGrid(buf)` → `ingestGrid(grid, name)`.
  - else → `readAsText` → `ingestGrid(parseCSV(text), name)`.
  - Preserve the existing multi-file `pending` counter / final `render()`.
- Add `.xlsx` + `application/vnd.openxmlformats-officedocument.spreadsheetml.sheet` to the `#file` `accept` attribute (`index.html:344`).

## Part B — `parseXlsxGrid(arrayBuffer)` → grid (new, no dependency)

Store-only ZIP reader + minimal sheet parser:

1. **Unzip (store-only):** read the End-Of-Central-Directory record (scan for signature `0x06054b50` from the tail), walk central-directory entries (`0x02014b50`), and for each, read the local file header (`0x04034b50`) to locate the stored bytes. **Compression method must be 0 (stored)**; if any needed part reports method 8 (deflate), throw a tagged error (see below). Decode part bytes as UTF-8.
2. Needed parts: `xl/worksheets/sheet1.xml` (required); `xl/sharedStrings.xml` (optional).
3. **Cell parse:** iterate `<row ...>` then `<c ...>` in document order. Column letter → 0-based index (`A`→0, `AA`→26). Cell value:
   - inline string: `<is><t...>TEXT</t></is>` (IC's format).
   - shared string: `t="s"` → integer index into sharedStrings `<si><t>…</t>` list.
   - otherwise the raw `<v>…</v>` text.
   Unescape XML entities (`&amp; &lt; &gt; &quot; &#39;`). Place values at their column index (fill gaps with `""`) so each row is a dense array; row 1 = headers. Return the grid.
4. **Excel-resave guard:** on a deflate part or missing `sheet1.xml`, throw `Error("XLSX_UNSUPPORTED")`; the caller shows: *"This .xlsx looks re-saved by another app. Please export a fresh Assets file from Inventory Check."* Other parse failures show a generic `"<file>: couldn't read this .xlsx."`.

`parseXlsxGrid` is pure (ArrayBuffer → grid) and independently testable in Node against an IC-produced file.

## Part C — Column detection & status decoding

Backward-compatible aliases (old names first so existing CSVs match unchanged):
- `detectAssetsCols` (`index.html:468`): `status: ["Check Status", "Status"]`, `note: ["Check Note", "Notes"]`, `added: ["Added Manually", "Added"]`.
- `detectBulkCols` unchanged (Parts/Bulk still CSV with old headers).
- `decodeStatus` (`index.html:516`): add `"accounted for"` → `"ok"` (keep `"accounted"`/`"match"`). Missing/Short → `bad`, Damaged/Over → `warn` already cover the new `Missing`/`Damaged`.
- `decodeAdded` already accepts `"yes"` → the new `Added` column works as-is.
- The cosmetic row color has no IM meaning and is ignored. `S/N` is unused by the scoreboard, but Part E adds a `ser` mapping so the round-trip export can echo it back — see Part E.

Note the `findCol` substring behavior: candidate `"Status"` also substring-matches `"Check Status"`, but `"Check Status"` is listed first and returns first, so old CSVs still bind to the right column. New xlsx (only `Status`) skips `Check Status` and binds `Status`. Same logic for note/added.

## Part D — Missing-by-default scoreboard (Assets only)

Under the new model every asset has a status, so blank-based "checked" is meaningless. For **assets**:
- **Completeness** (`renderCompleteness`, `index.html:938`): `complete = rows.filter(r._status === "ok").length` (Accounted For only); denominator = total. Label stays "Check completeness." Parts/Bulk keep `r._status`-truthy counting.
- **Exceptions** (`index.html:822`): Missing rows already decode to the `bad` bucket → show under "Missing." The `unchecked` bucket (`!r._status && !r._added`) is naturally empty for assets, so the "Not Checked" section renders nothing — acceptable; no special-case needed. (Parts/Bulk "Not Counted" unchanged.)
- **Breakdown** (`index.html:736`): Missing assets count as `miss`, so groups with outstanding assets stay in the discrepancy view — correct. `unchecked` per-group is 0 for assets. No change required beyond completeness.

Only `renderCompleteness` needs an assets branch; the rest falls out of Missing decoding to `bad`.

## Part E — Round-trip: corrected Assets export as colored `.xlsx`

Port IC's writer verbatim into IM (independent duplicate — the apps don't share modules): `CRC_TABLE`, `crc32`, `concatBytes`, `zipStore`, the `XLSX_*` OOXML constants, `xmlEsc`, `colLetter`, `xlsxCell`, `assetRowStyle`, `assetStatusText`, and a `buildAssetsXlsx()` adapted to IM's data:
- Rows from `state.tabs.assets.rows`; columns via `state.tabs.assets.cols` (`id/cat/make/model/market`, plus `S/N` from the asset's serial column if present — IM has no `ser` mapping today, so add `ser: ["Serial # or VIN","Serial #","Serial","VIN","S/N"]` to `detectAssetsCols` so the round-tripped file keeps serials; blank if absent).
- **Status/Added/Notes come from IM's reconciled edit state**: `assetStatusText(r._status)` (`ok→"Accounted For"`, `warn→"Damaged"`, else `"Missing"`), `r._added ? "Yes" : ""`, `r._note`. Row fill via `assetRowStyle` using `r._added`/`r._status`.

`exportCorrected` (`index.html:1251`) becomes format-aware:
- **assets** → `buildAssetsXlsx()` Blob, download `…-corrected.xlsx` (MIME `application/vnd.openxmlformats-officedocument.spreadsheetml.sheet`).
- **parts/bulk** → existing CSV path unchanged.
- Update the button label `#exportBtn` (`index.html:909`) from "Export Corrected CSV" to "Export Corrected" (it's `.xlsx` for assets now).

`STATUS_LABELS.assets` may read `ok: "Accounted For"` for editor/label consistency, but export text comes from `assetStatusText` to guarantee the exact IC contract.

## Part F — Versioning

Bump `APP_VERSION` `"0.6.0"` → `"0.7.0"` (`index.html:405`); `git tag v0.7.0`. Update README's version + the "How to use" note (it says "round-trip CSVs" — now `.xlsx` for assets).

## Testing (manual — single-file app, no framework)

1. Export a real Assets `.xlsx` from Inventory Check (v1.7.0), load it into IM → routes to Assets tab; stat tiles show Accounted/Missing/Damaged; **completeness = accounted / total**; Missing assets appear under the Missing exceptions.
2. Load an old Assets **CSV** (Check Status headers) → still imports (backward compat).
3. Load a Parts/Bulk CSV → unchanged (Match/Short/Over, Not Counted).
4. Fix a Missing asset to Accounted in IM → completeness rises; **Export Corrected** → `…-corrected.xlsx` downloads; opens in Google Sheets with 9 columns, correct colors, `Accounted For` text; **re-imports into Inventory Check** cleanly.
5. Re-save the IC `.xlsx` in Excel, load into IM → clear "export a fresh file from Inventory Check" message (no crash).
6. `parseXlsxGrid` Node check: extract from IM's `index.html`, feed an IC-produced file, assert grid headers + a couple cell values.
