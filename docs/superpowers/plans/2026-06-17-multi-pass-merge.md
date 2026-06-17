# Multi-Pass Merge Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans. Steps use checkbox (`- [ ]`) syntax.
>
> **Testing note:** This project is a single `index.html` with **no test runner** (vanilla, zero-build). Per project history (v0.3–v0.5) verification is: (a) `node --check` on the extracted script block for syntax, and (b) the concrete manual CSV scenarios given in each task. Do NOT add a test framework. The merge logic is factored into pure functions so it can be reasoned about directly.

**Goal:** Let Inventory Manager load multiple partial Inventory Check exports of the same check and reconcile them into one corrected dataset — auto-resolving "checked beats blank", flagging genuine conflicts, and de-duplicating added assets by Asset ID — then export the single source of truth.

**Architecture:** Replace the overwrite-on-load `applyImport` with a key-based `mergeImport` that folds same-key rows via a pure `reconcile()` function, tracking per-row provenance (`_sources`) and a `_conflict` flag. Conflicts and keyless added rows surface as two new sections in the existing exceptions panel and are resolved with the existing v0.5.0 item editor. Export serializes the reconciled `_status`/`_note`.

**Tech Stack:** Single-file vanilla JS + CSS in `index.html`. No dependencies, no build. Deploy = GitHub Pages from `main`/root.

**Spec:** `docs/superpowers/specs/2026-06-17-multi-pass-merge-design.md`

**Syntax-check command (used in every task):**
```bash
cd /home/cj/development/inventory-manager
awk '/^<script>/{f=1;next} /^<\/script>/{f=0} f' index.html > /tmp/im_check.js && node --check /tmp/im_check.js && echo "JS OK" && rm /tmp/im_check.js
```

---

## File map

Everything is in `index.html`:
- **`<script>` — Merge helpers (new):** `STATUS_SEVERITY`, `rowKey()`, `reconcile()`, `sourceLabel()` — pure.
- **`<script>` — Ingest:** `mergeImport()` replaces `applyImport()`; `sameCheckOK()` guard; `ingestFile` wires both.
- **`<script>` — Exceptions:** `buildExceptions()` gains `conflict`/`keyless` buckets; `renderExceptions()` renders two new sections; `renderRow()` shows conflict provenance.
- **`<script>` — Editor:** `openEdit()` gains conflict quick-picks + a Delete action for keyless rows.
- **`<script>` — Session/Export:** `renderSession()` shows "Merged N files / N to resolve"; `exportCorrected()` serializes reconciled values for all rows.
- **`<style>`:** small styles for the conflict source line + quick-pick buttons.
- Version constant + README.

State shape note: each tab is `{ headers, rows, cols, fileName }` today. Merge adds `tab.mergedFiles` (array of source labels). Per-row new fields: `_key`, `_sources` (`{srcLabel:{status,note}}`), `_conflict` (bool), `_keyless` (bool). Existing per-row `_uid/_status/_note/_added/_origStatus/_origNote/_edited` are unchanged.

---

### Task 1: Pure merge helpers

**Files:** Modify `index.html` — add a new block immediately AFTER the `statusVocab` definition (the `// ---------- Edit metadata ----------` section added in v0.5.0).

- [ ] **Step 1 — Add the helpers.** Insert:
```js
// ---------- Merge helpers (pure) ----------
// Higher = more severe; used to hold a conflict at its worst status until resolved.
const STATUS_SEVERITY = { bad: 3, warn: 2, ok: 1, "": 0 };

// Stable per-row key for matching the same physical item across passes.
// Assets: Asset ID. Parts/Bulk: PART # + Bin Location. "" = keyless (added, no id).
function rowKey(kind, r, cols) {
  const id = String((cols.id ? r[cols.id] : "") ?? "").trim().toLowerCase();
  if (kind === "assets") return id;
  if (!id) return "";
  const bin = String((cols.bin ? r[cols.bin] : "") ?? "").trim().toLowerCase();
  return `${id}␟${bin}`; // ␟ unit separator
}

// Reconcile a row's per-source answers into one {status, note, conflict}.
// Rule: checked beats blank; 2+ distinct non-blank statuses = conflict, held at the
// most-severe status. Note follows the winning status (else any non-empty note).
function reconcile(sources) {
  const entries = Object.values(sources);
  const distinct = [...new Set(entries.map(e => e.status).filter(s => s))];
  let status = "", conflict = false;
  if (distinct.length === 1) {
    status = distinct[0];
  } else if (distinct.length > 1) {
    conflict = true;
    status = distinct.slice().sort((a, b) => STATUS_SEVERITY[b] - STATUS_SEVERITY[a])[0];
  }
  let note = "";
  for (const e of entries) { if (e.status === status && e.note) { note = e.note; break; } }
  if (!note) { const wn = entries.find(e => e.note); if (wn) note = wn.note; }
  return { status, note, conflict };
}

// Human label for a per-source status in a conflict line ("" → "Unchecked").
function sourceLabel(kind, s) { return s ? statusVocab(kind)[s] : "Unchecked"; }
```

- [ ] **Step 2 — Syntax check.** Run the syntax-check command. Expected: `JS OK`.

- [ ] **Step 3 — Logic spot-check (temporary).** Append to `/tmp/im_check.js` a quick assert run to confirm `reconcile`, then delete it (do not leave it in the repo):
```bash
cd /home/cj/development/inventory-manager
awk '/^<script>/{f=1;next} /^<\/script>/{f=0} f' index.html > /tmp/im_check.js
cat >> /tmp/im_check.js <<'EOF'
const a = reconcile({f1:{status:"",note:""}, f2:{status:"bad",note:"x"}});
console.assert(a.status==="bad" && a.conflict===false && a.note==="x", "checked-beats-blank", a);
const b = reconcile({f1:{status:"bad",note:""}, f2:{status:"ok",note:""}});
console.assert(b.status==="bad" && b.conflict===true, "conflict-most-severe", b);
const c = reconcile({f1:{status:"ok",note:""}, f2:{status:"ok",note:""}});
console.assert(c.status==="ok" && c.conflict===false, "agreement", c);
console.log("reconcile checks done");
EOF
node /tmp/im_check.js && rm /tmp/im_check.js
```
Expected: `reconcile checks done` with no `Assertion failed` lines.

- [ ] **Step 4 — Commit.**
```bash
git add index.html && git commit -m "feat: pure merge helpers (rowKey, reconcile, severity)"
```

---

### Task 2: mergeImport replaces applyImport

**Files:** Modify `index.html` — replace the `applyImport` function and its three call sites in `ingestFile`.

Current `applyImport`:
```js
function applyImport(kind, headers, rows, fileName, cols) {
  state.tabs[kind].headers  = headers;
  state.tabs[kind].rows     = rows;
  state.tabs[kind].cols     = cols;
  state.tabs[kind].fileName = fileName;
}
```

- [ ] **Step 1 — Replace `applyImport` with `mergeImport`:**
```js
// Reconcile incoming rows into a tab by key. First file into an empty tab seeds it;
// later files fold matching keys (see reconcile) and append new / keyless rows.
function mergeImport(kind, headers, rows, fileName, cols) {
  const tab = state.tabs[kind];
  const seeding = tab.rows.length === 0;
  if (seeding) {
    tab.headers = headers;
    tab.cols = cols;
    tab.fileName = fileName;
    tab.mergedFiles = [];
  }
  // Distinct source label so duplicate filenames keep separate provenance.
  let src = fileName || "import";
  if (tab.mergedFiles.includes(src)) { let n = 2; while (tab.mergedFiles.includes(`${src} (${n})`)) n++; src = `${src} (${n})`; }
  tab.mergedFiles.push(src);

  const index = new Map();
  for (const r of tab.rows) if (r._key) index.set(r._key, r);

  for (const r of rows) {
    const key = rowKey(kind, r, cols);
    const entry = { status: r._status, note: r._note };
    if (!key) {                              // keyless added row — keep, no dedupe
      r._key = ""; r._keyless = true;
      r._sources = { [src]: entry }; r._conflict = false;
      r._origStatus = r._status; r._origNote = r._note; r._edited = false;
      tab.rows.push(r);
      continue;
    }
    const existing = index.get(key);
    if (!existing) {                         // new keyed row
      r._key = key; r._keyless = false;
      r._sources = { [src]: entry };
      const rec = reconcile(r._sources);
      r._status = rec.status; r._note = rec.note; r._conflict = rec.conflict;
      r._origStatus = r._status; r._origNote = r._note; r._edited = false;
      tab.rows.push(r); index.set(key, r);
    } else {                                 // fold into existing row
      existing._sources[src] = entry;
      const rec = reconcile(existing._sources);
      existing._status = rec.status; existing._note = rec.note; existing._conflict = rec.conflict;
      existing._origStatus = existing._status; existing._origNote = existing._note; existing._edited = false;
    }
  }
}
```

- [ ] **Step 2 — Wire the assets call site.** In `ingestFile`, replace:
```js
    applyImport("assets", headers, enriched, fileName, cols);
```
with:
```js
    if (!sameCheckOK("assets", headers, enriched, cols)) return;
    mergeImport("assets", headers, enriched, fileName, cols);
```
(`sameCheckOK` is added in Task 3; it returns `true` until then because the tab is empty on first load — but implement Task 3 before manual-testing multi-file merges.)

- [ ] **Step 3 — Wire the parts/bulk call sites.** Replace:
```js
  if (partsRows.length) applyImport("parts", headers, partsRows, fileName, cols);
  if (bulkRows.length)  applyImport("bulk",  headers, bulkRows,  fileName, cols);
```
with:
```js
  if (partsRows.length && sameCheckOK("parts", headers, partsRows, cols)) mergeImport("parts", headers, partsRows, fileName, cols);
  if (bulkRows.length  && sameCheckOK("bulk",  headers, bulkRows,  cols)) mergeImport("bulk",  headers, bulkRows,  fileName, cols);
```

- [ ] **Step 4 — Syntax check.** Run the syntax-check command → `JS OK`. (It will reference `sameCheckOK`; that's fine for `node --check` since it's a function declaration added in Task 3 — but do Task 3 next before loading in a browser.)

- [ ] **Step 5 — Commit.**
```bash
git add index.html && git commit -m "feat: mergeImport reconciles same-key rows across passes"
```

---

### Task 3: Same-check guard

**Files:** Modify `index.html` — add `sameCheckOK` just above `mergeImport`.

- [ ] **Step 1 — Add the guard:**
```js
// Guard against fusing two different checks. Returns true to proceed; on a mismatch
// asks the user to confirm. Always true when the target tab is empty (first load).
function sameCheckOK(kind, headers, rows, cols) {
  const tab = state.tabs[kind];
  if (!tab.rows.length) return true;
  const sameCols = tab.cols.id === cols.id && tab.cols.status === cols.status
    && (kind === "assets" || tab.cols.bin === cols.bin);
  const existingKeys = new Set(tab.rows.filter(r => r._key).map(r => r._key));
  let hit = 0, total = 0;
  for (const r of rows) { const k = rowKey(kind, r, cols); if (k) { total++; if (existingKeys.has(k)) hit++; } }
  const overlap = total ? hit / total : 1;
  if (sameCols && overlap >= 0.10) return true;
  return confirm(`This file looks like a different ${kind} check than what's already loaded `
    + `(${Math.round(overlap * 100)}% key overlap${sameCols ? "" : ", different columns"}). Merge anyway?`);
}
```

- [ ] **Step 2 — Syntax check** → `JS OK`.

- [ ] **Step 3 — Manual scenario A (basic two-pass merge).** Create two assets CSVs and load BOTH together via Load CSV.
  `passA.csv`:
  ```csv
  Asset ID,Equipment Class,Check Status,Check Note,Added Manually
  A1,Excavators,Accounted,,
  A2,Excavators,Missing,not on lot,
  A3,Loaders,,,
  ```
  `passB.csv`:
  ```csv
  Asset ID,Equipment Class,Check Status,Check Note,Added Manually
  A1,Excavators,,,
  A2,Excavators,,,
  A3,Loaders,Accounted,,
  ```
  Expected: Total **3** (not 6 — rows merged by Asset ID). Accounted **2** (A1, A3), Missing **1** (A2). Completeness **3/3 = 100%**. Session strip shows "Merged 2 files". No conflicts.

- [ ] **Step 4 — Manual scenario B (different-check guard).** Load `passA.csv`, then load a file with non-overlapping IDs (`Z9,Cranes,Accounted,,`). Expected: a confirm() dialog warns of low overlap; Cancel leaves the assets tab unchanged.

- [ ] **Step 5 — Commit.**
```bash
git add index.html && git commit -m "feat: same-check guard before merging a second file"
```

---

### Task 4: Conflict + keyless sections in the exceptions panel

**Files:** Modify `index.html` — `buildExceptions`, `renderRow`, `renderExceptions`.

- [ ] **Step 1 — buildExceptions: add buckets + routing.** Replace the bucket init + loop:
```js
  const buckets = { added: [], bad: [], warn: [], unchecked: [] };
  for (const r of tab.rows) {
    if (!matches(r)) continue;
    if (r._added) buckets.added.push(r);
    if (r._status === "bad" || r._status === "warn") buckets[r._status].push(r);
    else if (!r._status && !r._added) buckets.unchecked.push(r);
  }
```
with:
```js
  const buckets = { conflict: [], keyless: [], added: [], bad: [], warn: [], unchecked: [] };
  for (const r of tab.rows) {
    if (!matches(r)) continue;
    if (r._conflict) { buckets.conflict.push(r); continue; }   // resolve first, shown alone
    if (r._keyless)  { buckets.keyless.push(r);  continue; }   // added, no id → review
    if (r._added) buckets.added.push(r);
    if (r._status === "bad" || r._status === "warn") buckets[r._status].push(r);
    else if (!r._status && !r._added) buckets.unchecked.push(r);
  }
```

- [ ] **Step 2 — buildExceptions: sort the new buckets.** After the existing `buckets.added.sort(...); buckets.bad.sort(...); buckets.warn.sort(...); buckets.unchecked.sort(...);` lines, add:
```js
  buckets.conflict.sort(sortByGroup);
  buckets.keyless.sort(sortByGroup);
```

- [ ] **Step 3 — renderRow: show conflict provenance.** In `renderRow`, just after the `const editedTag = ...` line, add:
```js
    const conflictLine = r._conflict
      ? `<div class="conflict-line">⚠ ${Object.entries(r._sources || {})
          .map(([f, e]) => `${escapeHtml(f)}: ${escapeHtml(sourceLabel(kind, e.status))}`).join(" · ")}</div>`
      : "";
```
Then add `${conflictLine}` inside BOTH returned `.ex-row` templates — immediately before the closing `</div>` of each (after the existing `${r._note ? ...}` line in each branch).

- [ ] **Step 4 — renderExceptions: total + sections + labels.** Update the total line:
```js
  const totalEx = buckets.added.length + buckets.bad.length + buckets.warn.length + buckets.unchecked.length;
```
to:
```js
  const totalEx = buckets.conflict.length + buckets.keyless.length + buckets.added.length
                + buckets.bad.length + buckets.warn.length + buckets.unchecked.length;
```
Update the `$("#exceptions").innerHTML = ...` assignment to prepend the two new sections:
```js
  $("#exceptions").innerHTML =
      renderSection("conflict", "Conflicts", "conflict", buckets.conflict)
    + renderSection("keyless", "Needs review — added", "keyless", buckets.keyless)
    + renderSection("added", "Added Manually", "added", buckets.added, { hideAddedTag: true })
    + renderSection("bad",   labels.bad,       isAssets ? "missing" : "short", buckets.bad)
    + renderSection("unchecked", labels.unchecked, "unchecked", buckets.unchecked)
    + renderSection("warn",  labels.warn,      isAssets ? "damaged" : "over",  buckets.warn);
```
(Conflicts and keyless default to expanded — do not add them to the `state.collapsed` defaults block, which only collapses `warn`/`added`.)

- [ ] **Step 5 — Add section colors (CSS).** After the `.ex-section.unchecked .ex-section-head { color: var(--muted); }` rule add:
```css
  .ex-section.conflict .ex-section-head { color: var(--bad); }
  .ex-section.keyless  .ex-section-head { color: var(--warn); }
  .conflict-line { color: var(--bad); font-size: 12px; margin-top: 4px; }
```

- [ ] **Step 6 — Syntax check** → `JS OK`.

- [ ] **Step 7 — Manual scenario C (conflict).** Load `passA.csv` (Task 3) then `passConflict.csv`:
  ```csv
  Asset ID,Equipment Class,Check Status,Check Note,Added Manually
  A1,Excavators,Missing,gone,
  ```
  Expected: a **Conflicts (1)** section at the top showing `A1` with a line like `⚠ passA.csv: Accounted · passConflict.csv: Missing`. A1 is held at the most-severe (Missing) so it also counts toward the Missing tile. It does NOT also appear in the Missing section (conflict rows are shown only in Conflicts).

- [ ] **Step 8 — Commit.**
```bash
git add index.html && git commit -m "feat: Conflicts + Needs-review-added exception sections"
```

---

### Task 5: Resolve conflicts + delete keyless rows in the editor

**Files:** Modify `index.html` — `openEdit`.

- [ ] **Step 1 — Conflict quick-picks + clear flag on save + keyless Delete.** In `openEdit`'s `renderBox`, replace the `$("#editBox").innerHTML = ...` template's button area and add wiring. Specifically:

  (a) Add a source quick-pick block (only when conflicted) right after the `<div class="seg-row">${btns}</div>` line:
```js
      ${r._conflict ? `<div class="small" style="margin:8px 0 4px">From passes</div>
        <div class="seg-row">${Object.entries(r._sources || {}).map(([f, e]) =>
          `<button class="seg" data-st="${e.status}">${escapeHtml(f)}: ${escapeHtml(sourceLabel(kind, e.status))}</button>`).join("")}</div>` : ""}
```
  (The quick-pick buttons reuse the same `.seg[data-st]` handler already wired, so picking one stages that status.)

  (b) Add a Delete button for keyless rows, right after the `${r._edited ? ... Revert ...}` line:
```js
      ${r._keyless ? `<button id="editDelete" class="btn-ghost" style="width:100%;margin-top:8px;color:var(--bad)">Delete this added item</button>` : ""}
```

  (c) In the Save handler, set `r._conflict = false;` so saving resolves the conflict. Change:
```js
    $("#editSave").addEventListener("click", () => {
      r._status = pending;
      r._note = $("#editNote").value.trim();
      r._edited = (r._status !== r._origStatus) || (r._note !== r._origNote);
      closeEdit();
      render();
    });
```
to:
```js
    $("#editSave").addEventListener("click", () => {
      r._status = pending;
      r._note = $("#editNote").value.trim();
      r._edited = (r._status !== r._origStatus) || (r._note !== r._origNote);
      r._conflict = false;                       // a decision resolves the conflict
      closeEdit();
      render();
    });
```

  (d) After the Revert wiring block, add the Delete wiring:
```js
    const del = $("#editDelete");
    if (del) del.addEventListener("click", () => {
      const tab = state.tabs[state.active];
      tab.rows = tab.rows.filter(x => x !== r);
      closeEdit();
      render();
    });
```

- [ ] **Step 2 — Syntax check** → `JS OK`.

- [ ] **Step 3 — Manual scenario D (resolve conflict).** Continue scenario C: tap the `A1` conflict row → editor shows the status segments AND "From passes" quick-picks (`passA.csv: Accounted`, `passConflict.csv: Missing`). Pick one → Save. Expected: Conflicts section drops to 0, A1 moves to the chosen status everywhere, row shows the ✏ edited tag.

- [ ] **Step 4 — Manual scenario E (keyless delete).** Build an added-no-id row by loading:
  ```csv
  Asset ID,Equipment Class,Check Status,Check Note,Added Manually
  ,Attachments,Accounted,found loose,Yes
  ```
  Expected: it appears under **Needs review — added (1)**. Tap it → editor shows a red "Delete this added item" button → Delete removes it; the section disappears.

- [ ] **Step 5 — Commit.**
```bash
git add index.html && git commit -m "feat: resolve conflicts via quick-picks; delete keyless added rows"
```

---

### Task 6: Session strip — merged count + to-resolve count

**Files:** Modify `index.html` — `renderSession`.

- [ ] **Step 1 — Add the counts.** In `renderSession`, after the `const editedNote = ...` line add:
```js
  const mergedCount = (tab.mergedFiles || []).length;
  const mergedNote = mergedCount > 1 ? `<span style="font-weight:600">Merged ${mergedCount} files</span>` : "";
  const toResolve = tab.rows.filter(r => r._conflict).length;
  const reviewCount = tab.rows.filter(r => r._keyless).length;
  const flagNote = (toResolve || reviewCount)
    ? `<span style="color:var(--bad);font-weight:600">${[
        toResolve ? `${toResolve} to resolve` : "",
        reviewCount ? `${reviewCount} added to review` : "",
      ].filter(Boolean).join(" · ")}</span>`
    : "";
```
Then add `${mergedNote}` and `${flagNote}` into the `sess.innerHTML` template (e.g. right after the `${editedNote}` line).

- [ ] **Step 2 — Syntax check** → `JS OK`.

- [ ] **Step 3 — Manual verify.** Reload scenario C (two files, one conflict): session strip reads `Merged 2 files` and `1 to resolve`; after resolving (scenario D) the `to resolve` note disappears.

- [ ] **Step 4 — Commit.**
```bash
git add index.html && git commit -m "feat: session strip shows merged-file + to-resolve counts"
```

---

### Task 7: Export reconciled values; version + README

**Files:** Modify `index.html` (`exportCorrected`, `APP_VERSION`), `README.md`.

- [ ] **Step 1 — Export serializes reconciled status/note for ALL rows.** In `exportCorrected`, replace the cell-mapping:
```js
    const cells = tab.headers.map(h => {
      if (r._edited && cols.status && h === cols.status) return csvCell(vocab[r._status] ?? "");
      if (r._edited && cols.note   && h === cols.note)   return csvCell(r._note);
      return csvCell(r[h]);
    });
```
with (drop the `_edited` condition — the merged `_status`/`_note` is the source of truth, so every row's status/note cell reflects it; a merged "checked-beats-blank" row would otherwise export its blank original cell and lose the merge):
```js
    const cells = tab.headers.map(h => {
      if (cols.status && h === cols.status) return csvCell(vocab[r._status] ?? "");
      if (cols.note   && h === cols.note)   return csvCell(r._note);
      return csvCell(r[h]);
    });
```

- [ ] **Step 2 — Bump version.** Change `const APP_VERSION = "0.5.0";` to:
```js
const APP_VERSION = "0.6.0";
```

- [ ] **Step 3 — README.** Bump `**Version:** v0.5.0` → `v0.6.0`. Under "What it does", add a bullet:
```markdown
- **Multi-pass merge**: load several partial exports of the *same* check at once and Inventory Manager reconciles them into one — a checked item beats a blank one, genuine disagreements surface in a **Conflicts** section to resolve, and manually-added items are de-duplicated by Asset ID (keyless adds go to a **Needs review** section). Export the reconciled result as the single source of truth.
```
And amend the "Why no persistence" section's first sentence to note that loading multiple files now *reconciles on load* rather than the last file winning.

- [ ] **Step 4 — Syntax check** → `JS OK`.

- [ ] **Step 5 — Manual scenario F (export round-trip).** Run scenario A (two-pass merge → A2 Missing, A1/A3 Accounted), then **Export Corrected CSV**. Open the file: one row per Asset ID; `Check Status` column reads `Accounted`/`Missing`/`Accounted` reflecting the merge (NOT the blanks from either source). Re-load that exported file into a fresh page → Total 3, Accounted 2, Missing 1.

- [ ] **Step 6 — Commit.**
```bash
git add index.html README.md && git commit -m "v0.6.0: export reconciled values; multi-pass merge docs"
```

---

### Task 8: Tag, push, confirm live

- [ ] **Step 1 — Tag + push.**
```bash
cd /home/cj/development/inventory-manager
git tag v0.6.0 && git push origin main --tags
```

- [ ] **Step 2 — Confirm live** (GitHub Pages, ~1–2 min build lag):
```bash
for i in $(seq 1 20); do
  st=$(gh api repos/CJ-Rutter/inventory-manager/pages/builds/latest --jq .status 2>/dev/null)
  ver=$(curl -s "https://cj-rutter.github.io/inventory-manager/index.html?cb=$(date +%s%N)" | grep -o 'APP_VERSION = "[0-9.]*"' | head -1)
  echo "[$i] build=$st live=$ver"; echo "$ver" | grep -q '0.6.0' && { echo ">>> v0.6.0 LIVE"; break; }; sleep 15
done
```

---

## Self-Review

**Spec coverage:**
- New flow / reconcile-on-load → Task 2 (`mergeImport`) ✅
- Row keys (Asset ID; PART#+Bin) → Task 1 (`rowKey`) ✅
- Same-check guard (columns + <10% overlap) → Task 3 ✅
- Reconcile rule (checked-beats-blank; conflict at most-severe) → Task 1 (`reconcile`) ✅
- Added auto-merge on Asset ID → falls out of keyed merge (an added row with an ID keys normally) Task 2 ✅; keyless added → review section Task 4 + delete Task 5 ✅
- Provenance `_sources` + conflict view → Task 2 (capture) + Task 4 (render) + Task 5 (quick-picks) ✅
- Conflicts + Needs-review sections; resolve via existing editor → Task 4 + Task 5 ✅
- Session strip "Merged N / N to resolve" → Task 6 ✅
- Export reconciled source of truth → Task 7 ✅
- Version/docs/deploy → Task 7 + Task 8 ✅

**Placeholder scan:** none — every step has concrete code/commands and explicit expected output.

**Type/name consistency:** `rowKey(kind,r,cols)`, `reconcile(sources)→{status,note,conflict}`, `sourceLabel(kind,s)`, `STATUS_SEVERITY`, `mergeImport(kind,headers,rows,fileName,cols)`, `sameCheckOK(kind,headers,rows,cols)`, per-row `_key/_sources/_conflict/_keyless`, `tab.mergedFiles`, bucket names `conflict`/`keyless` (section keys match `renderSection` calls + CSS `.ex-section.conflict`/`.keyless` + collapse-defaults exclusion). `exportCorrected` uses `vocab`/`statusVocab` already in scope. `applyImport` fully removed (no dangling references — all three call sites replaced in Task 2).

**Risk note:** Export now normalizes status casing on every row (was edited-only in v0.5.0) — intentional and required for merge correctness; re-import decodes case-insensitively so it's safe.
