# Inventory Manager — Consume Assets `.xlsx` Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Make Inventory Manager import Inventory Check v1.7.0's new colored Assets `.xlsx` (backward-compatible with old CSV), reflect Missing-by-default in the scoreboard, and export corrected assets back as the same colored `.xlsx`.

**Architecture:** Single-file `index.html`, no build, no framework, no dependencies. A from-scratch store-only ZIP reader turns the `.xlsx` into the same grid `parseCSV` produces, feeding the existing ingest/merge pipeline. IC's xlsx writer is ported verbatim for the round-trip export (intentional duplication across two independent apps).

**Tech Stack:** Vanilla HTML/CSS/JS. Verification is manual-in-browser plus Node for the pure parser/writer functions.

**Spec:** `docs/superpowers/specs/2026-07-02-consume-assets-xlsx-import-export-design.md`

**Status codes (existing):** `"ok"`=Accounted, `"bad"`=Missing/Short, `"warn"`=Damaged/Over, `""`=blank. New IC `.xlsx` always populates Status (Missing default → decodes to `"bad"`).

---

### Task 1: Version bump + README note

**Files:** Modify `index.html:405`, `README.md`

- [ ] **Step 1: Bump `APP_VERSION`**

Change `index.html:405` from:
```js
const APP_VERSION = "0.6.0";
```
to:
```js
const APP_VERSION = "0.7.0";
```

- [ ] **Step 2: Update README version + round-trip note**

In `README.md`, change the version line `**Version:** v0.6.0` to `**Version:** v0.7.0`. Then find the "How to use" step mentioning round-trip CSVs (it reads: `the round-trip CSVs with \`Check Status\` / \`Counted Qty\` / \`Variance\` columns`) and change it to:
```
the files exported from Inventory Check (Assets as `.xlsx`; Parts/Bulk as `.csv` with `Check Status` / `Counted Qty` / `Variance` columns)
```

- [ ] **Step 3: Verify**

Open `index.html` in a browser; footer reads `v0.7.0`. `grep -n 'APP_VERSION = "0.7.0"' index.html` → 1 hit; `grep -c 'v0.6.0' README.md` → 0.

- [ ] **Step 4: Commit**

```bash
git add index.html README.md
git commit -m "chore: bump Inventory Manager to v0.7.0"
```

---

### Task 2: Column aliases + status decoding + serial mapping

**Files:** Modify `index.html:468-480` (`detectAssetsCols`), `index.html:516-522` (`decodeStatus`)

- [ ] **Step 1: Add header aliases + `ser` to `detectAssetsCols`**

Change `index.html:468-480` from:
```js
function detectAssetsCols(headers) {
  return {
    id:      findCol(headers, ["Asset ID", "Asset #", "AssetNumber", "Asset", "Unit #"]),
    cat:     findCol(headers, ["Equipment Class", "Cat Class", "Category", "Class"]),
    make:    findCol(headers, ["Make"]),
    model:   findCol(headers, ["Model"]),
    desc:    findCol(headers, ["Description"]),
    market:  findCol(headers, ["Market", "Branch"]),
    status:  findCol(headers, ["Check Status"]),
    note:    findCol(headers, ["Check Note"]),
    added:   findCol(headers, ["Added Manually"]),
  };
}
```
to:
```js
function detectAssetsCols(headers) {
  return {
    id:      findCol(headers, ["Asset ID", "Asset #", "AssetNumber", "Asset", "Unit #"]),
    cat:     findCol(headers, ["Equipment Class", "Cat Class", "Category", "Class"]),
    make:    findCol(headers, ["Make"]),
    model:   findCol(headers, ["Model"]),
    ser:     findCol(headers, ["Serial # or VIN", "Serial #", "Serial", "VIN", "S/N"]),
    desc:    findCol(headers, ["Description"]),
    market:  findCol(headers, ["Market", "Branch"]),
    // Old CSV names first so existing exports bind to them; new .xlsx uses the short names.
    status:  findCol(headers, ["Check Status", "Status"]),
    note:    findCol(headers, ["Check Note", "Notes"]),
    added:   findCol(headers, ["Added Manually", "Added"]),
  };
}
```

- [ ] **Step 2: Teach `decodeStatus` the new "Accounted For" label**

Change `index.html:516-522` from:
```js
function decodeStatus(raw) {
  const s = String(raw || "").trim().toLowerCase();
  if (s === "accounted" || s === "match") return "ok";
  if (s === "missing"   || s === "short") return "bad";
  if (s === "damaged"   || s === "over")  return "warn";
  return "";
}
```
to:
```js
function decodeStatus(raw) {
  const s = String(raw || "").trim().toLowerCase();
  if (s === "accounted" || s === "accounted for" || s === "match") return "ok";
  if (s === "missing"   || s === "short") return "bad";
  if (s === "damaged"   || s === "over")  return "warn";
  return "";
}
```

- [ ] **Step 3: Verify**

`grep -n '"Check Status", "Status"' index.html` → 1 hit; `grep -n 'accounted for' index.html` → 1 hit; `grep -n 'ser:      findCol' index.html` → 1 hit.

- [ ] **Step 4: Commit**

```bash
git add index.html
git commit -m "feat(assets): accept new xlsx headers (Status/Notes/Added) + Accounted For; map serial"
```

---

### Task 3: `parseXlsxGrid` — store-only xlsx reader (pure, node-tested)

Insert a new block **immediately before** `function ingestFile` (`index.html:581`). Find it: `grep -n 'function ingestFile' index.html`, insert above it.

**Files:** Modify `index.html` (insert before `ingestFile`)

- [ ] **Step 1: Insert the reader**

```js
// ---------- .xlsx reader: store-only unzip + inline/shared strings → grid ----------
// Reads Inventory Check's store-only (uncompressed) .xlsx. A file re-saved by
// Excel/Sheets is DEFLATE-compressed and throws XLSX_UNSUPPORTED (we can't inflate).
function xlsxReadParts(buf) {
  const bytes = new Uint8Array(buf);
  const dv = new DataView(buf);
  const dec = new TextDecoder("utf-8");
  let eocd = -1;
  for (let i = bytes.length - 22; i >= 0; i--) {
    if (dv.getUint32(i, true) === 0x06054b50) { eocd = i; break; }
  }
  if (eocd < 0) throw new Error("XLSX_UNSUPPORTED");
  const cdCount = dv.getUint16(eocd + 10, true);
  let off = dv.getUint32(eocd + 16, true);
  const parts = {};
  for (let n = 0; n < cdCount; n++) {
    if (dv.getUint32(off, true) !== 0x02014b50) throw new Error("XLSX_UNSUPPORTED");
    const nameLen = dv.getUint16(off + 28, true);
    const extraLen = dv.getUint16(off + 30, true);
    const commentLen = dv.getUint16(off + 32, true);
    const compSize = dv.getUint32(off + 20, true);
    const localOff = dv.getUint32(off + 42, true);
    const name = dec.decode(bytes.subarray(off + 46, off + 46 + nameLen));
    if (dv.getUint32(localOff, true) !== 0x04034b50) throw new Error("XLSX_UNSUPPORTED");
    const lMethod = dv.getUint16(localOff + 8, true);
    const lNameLen = dv.getUint16(localOff + 26, true);
    const lExtraLen = dv.getUint16(localOff + 28, true);
    const dataStart = localOff + 30 + lNameLen + lExtraLen;
    if (lMethod !== 0) {
      if (name === "xl/worksheets/sheet1.xml" || name === "xl/sharedStrings.xml") throw new Error("XLSX_UNSUPPORTED");
    } else {
      parts[name] = dec.decode(bytes.subarray(dataStart, dataStart + compSize));
    }
    off += 46 + nameLen + extraLen + commentLen;
  }
  return parts;
}
function xmlUnescape(s) {
  return String(s).replace(/&lt;/g, "<").replace(/&gt;/g, ">").replace(/&quot;/g, '"')
    .replace(/&#39;/g, "'").replace(/&apos;/g, "'").replace(/&amp;/g, "&");
}
function colIndexFromRef(ref) {
  const m = /^([A-Z]+)/.exec(ref);
  if (!m) return 0;
  let n = 0;
  for (const ch of m[1]) n = n * 26 + (ch.charCodeAt(0) - 64);
  return n - 1;
}
function parseXlsxGrid(buf) {
  const parts = xlsxReadParts(buf);
  const sheet = parts["xl/worksheets/sheet1.xml"];
  if (!sheet) throw new Error("XLSX_UNSUPPORTED");
  let shared = [];
  if (parts["xl/sharedStrings.xml"]) {
    shared = (parts["xl/sharedStrings.xml"].match(/<si>[\s\S]*?<\/si>/g) || []).map(si =>
      (si.match(/<t[^>]*>([\s\S]*?)<\/t>/g) || [])
        .map(x => xmlUnescape(x.replace(/<t[^>]*>/, "").replace(/<\/t>/, ""))).join(""));
  }
  const grid = [];
  const rows = sheet.match(/<row[^>]*>[\s\S]*?<\/row>|<row[^>]*\/>/g) || [];
  for (const rowXml of rows) {
    const cells = [];
    const cMatches = rowXml.match(/<c\b[^>]*>[\s\S]*?<\/c>|<c\b[^>]*\/>/g) || [];
    for (const cXml of cMatches) {
      const refM = /r="([A-Z]+\d+)"/.exec(cXml);
      const idx = refM ? colIndexFromRef(refM[1]) : cells.length;
      const typeM = /t="([^"]+)"/.exec(cXml);
      const type = typeM ? typeM[1] : "";
      let val = "";
      if (type === "inlineStr") {
        const tM = /<t[^>]*>([\s\S]*?)<\/t>/.exec(cXml);
        val = tM ? xmlUnescape(tM[1]) : "";
      } else if (type === "s") {
        const vM = /<v>([\s\S]*?)<\/v>/.exec(cXml);
        val = vM ? (shared[Number(vM[1])] ?? "") : "";
      } else {
        const vM = /<v>([\s\S]*?)<\/v>/.exec(cXml);
        val = vM ? xmlUnescape(vM[1]) : "";
      }
      while (cells.length < idx) cells.push("");
      cells[idx] = val;
    }
    grid.push(cells);
  }
  return grid;
}
```

- [ ] **Step 2: Node round-trip test (no browser)**

Create `/tmp/claude-1000/-home-cj-development-nebula-sentinel/24251f1e-2a18-4be2-8fe2-4de42df445d0/scratchpad/xlsx-read-check.mjs`. In it: (a) paste copies of `crc32`+`CRC_TABLE`, `concatBytes`, `zipStore` from Inventory Check (`/home/cj/development/inventory-check/index.html` — grep them out), (b) build a store-only `.xlsx` whose `xl/worksheets/sheet1.xml` has one header row `Asset ID, Status, Notes` and one data row `1001, Accounted For, ok` using inline strings (write the exact XML by hand, wrap with the minimal `[Content_Types].xml`/`_rels/.rels`/`workbook.xml`/`workbook.xml.rels`/`styles.xml` from IC or just the parts needed — only `sheet1.xml` is required by the reader), (c) paste copies of `xlsxReadParts`/`xmlUnescape`/`colIndexFromRef`/`parseXlsxGrid`, (d) call `parseXlsxGrid(zipBytes.buffer)` and assert `grid[0]` deep-equals `["Asset ID","Status","Notes"]` and `grid[1]` equals `["1001","Accounted For","ok"]`. Also build a version with `t="inlineStr"` for the header and confirm it round-trips.

Run: `node /tmp/.../scratchpad/xlsx-read-check.mjs` → prints `READ_OK` on success. If it throws or asserts fail, fix `parseXlsxGrid` and re-run — do NOT commit a broken reader. (Scratch file is not committed.)

Alternatively, if the earlier IC fixture `/tmp/.../scratchpad/assets-test.xlsx` still exists, also parse it and assert `grid[0]` equals the 9 IC headers `["Asset ID","Cat Class","S/N","Make","Model","Market/Branch","Status","Added","Notes"]`.

- [ ] **Step 3: Commit**

```bash
git add index.html
git commit -m "feat(import): add store-only .xlsx reader (parseXlsxGrid)"
```

---

### Task 4: Route ingest through a grid + dispatch by file type

**Files:** Modify `index.html:581` (`ingestFile`), `index.html:1300-1315` (`#file` handler), `index.html:344` (`accept`)

- [ ] **Step 1: Split `ingestFile` into `ingestGrid`**

Change the first two lines of `ingestFile` (`index.html:581-582`) from:
```js
function ingestFile(text, fileName) {
  const grid = parseCSV(text);
```
to:
```js
function ingestFile(text, fileName) {
  ingestGrid(parseCSV(text), fileName);
}
function ingestGrid(grid, fileName) {
```
(This renames the old body to `ingestGrid(grid, fileName)` and leaves a thin `ingestFile` wrapper for the CSV path. The rest of the function body — `if (!grid.length) ...` onward — is unchanged and now belongs to `ingestGrid`.)

- [ ] **Step 2: Dispatch by extension in the `#file` handler**

Change `index.html:1300-1315` from:
```js
$("#file").addEventListener("change", (ev) => {
  const files = Array.from(ev.target.files || []);
  if (!files.length) return;
  let pending = files.length;
  for (const f of files) {
    const fr = new FileReader();
    fr.onload = () => {
      ingestFile(String(fr.result || ""), f.name);
      if (--pending === 0) {
        ev.target.value = "";
        render();
      }
    };
    fr.readAsText(f);
  }
});
```
to:
```js
$("#file").addEventListener("change", (ev) => {
  const files = Array.from(ev.target.files || []);
  if (!files.length) return;
  let pending = files.length;
  const done = () => { if (--pending === 0) { ev.target.value = ""; render(); } };
  for (const f of files) {
    const fr = new FileReader();
    const isXlsx = /\.xlsx$/i.test(f.name);
    fr.onload = () => {
      try {
        if (isXlsx) ingestGrid(parseXlsxGrid(fr.result), f.name);
        else ingestGrid(parseCSV(String(fr.result || "")), f.name);
      } catch (e) {
        if (e && e.message === "XLSX_UNSUPPORTED") {
          alert(`${f.name}: this .xlsx looks re-saved by another app. Please export a fresh Assets file from Inventory Check.`);
        } else {
          alert(`${f.name}: couldn't read this file.`);
        }
      }
      done();
    };
    if (isXlsx) fr.readAsArrayBuffer(f);
    else fr.readAsText(f);
  }
});
```

- [ ] **Step 3: Add `.xlsx` to the file input `accept`**

Change `index.html:344` from:
```html
    <input type="file" id="file" accept=".csv,text/csv" multiple />
```
to:
```html
    <input type="file" id="file" accept=".csv,text/csv,.xlsx,application/vnd.openxmlformats-officedocument.spreadsheetml.sheet" multiple />
```

- [ ] **Step 4: Verify (browser)**

Reload `index.html`. Load a real Assets `.xlsx` exported from Inventory Check → it routes to the Assets tab and shows rows/stats. Load a Parts/Bulk `.csv` → still works. `grep -n 'function ingestGrid' index.html` → 1 hit; `grep -n 'parseXlsxGrid(fr.result)' index.html` → 1 hit.

- [ ] **Step 5: Commit**

```bash
git add index.html
git commit -m "feat(import): dispatch .xlsx vs .csv by extension into shared ingestGrid"
```

---

### Task 5: Missing-by-default completeness (Assets)

**Files:** Modify `index.html:938-950` (`renderCompleteness`)

- [ ] **Step 1: Count only Accounted as complete for assets**

Change `index.html:938-950` from:
```js
function renderCompleteness() {
  const tab = state.tabs[state.active];
  const total = tab.rows.length;
  const checked = tab.rows.filter(r => r._status).length;
  const pct = total ? (checked / total) * 100 : 0;
  $("#completeness").innerHTML = `
    <div class="label">
      <span>Check completeness</span>
      <span class="val">${checked.toLocaleString()} / ${total.toLocaleString()} · ${formatPct(pct)}</span>
    </div>
    <div class="bar"><div class="fill" style="width:${pct}%"></div></div>
  `;
}
```
to:
```js
function renderCompleteness() {
  const kind = state.active;
  const tab = state.tabs[kind];
  const total = tab.rows.length;
  // Assets default to Missing (never blank), so "checked" = Accounted only.
  // Parts/Bulk keep any non-blank status as checked.
  const checked = kind === "assets"
    ? tab.rows.filter(r => r._status === "ok").length
    : tab.rows.filter(r => r._status).length;
  const label = kind === "assets" ? "Accounted" : "Check completeness";
  const pct = total ? (checked / total) * 100 : 0;
  $("#completeness").innerHTML = `
    <div class="label">
      <span>${label}</span>
      <span class="val">${checked.toLocaleString()} / ${total.toLocaleString()} · ${formatPct(pct)}</span>
    </div>
    <div class="bar"><div class="fill" style="width:${pct}%"></div></div>
  `;
}
```

- [ ] **Step 2: Verify (browser)**

Load an Assets `.xlsx` where some are Accounted and the rest Missing → the completeness bar reads `<accounted> / <total>` labeled "Accounted", not 100%. Load a Parts CSV → bar still labeled "Check completeness" and counts any status as checked.

- [ ] **Step 3: Commit**

```bash
git add index.html
git commit -m "feat(assets): completeness counts Accounted only under Missing-default"
```

---

### Task 6: Port Inventory Check's xlsx writer (CRC + ZIP + OOXML)

Insert this block **immediately before** `function exportCorrected` (`index.html:1251`). Find it: `grep -n 'function exportCorrected' index.html`, insert above it. This is a verbatim copy of Inventory Check's writer (independent duplicate).

**Files:** Modify `index.html` (insert before `exportCorrected`)

- [ ] **Step 1: Insert CRC-32 + store-only ZIP**

```js
// ---------- .xlsx writer (ported from Inventory Check; independent copy) ----------
const CRC_TABLE = (() => {
  const t = new Uint32Array(256);
  for (let n = 0; n < 256; n++) {
    let c = n;
    for (let k = 0; k < 8; k++) c = (c & 1) ? (0xEDB88320 ^ (c >>> 1)) : (c >>> 1);
    t[n] = c >>> 0;
  }
  return t;
})();
function crc32(bytes) {
  let c = 0xFFFFFFFF;
  for (let i = 0; i < bytes.length; i++) c = CRC_TABLE[(c ^ bytes[i]) & 0xFF] ^ (c >>> 8);
  return (c ^ 0xFFFFFFFF) >>> 0;
}
function concatBytes(parts) {
  let len = 0;
  for (const p of parts) len += p.length;
  const out = new Uint8Array(len);
  let o = 0;
  for (const p of parts) { out.set(p, o); o += p.length; }
  return out;
}
function zipStore(files) {
  const enc = new TextEncoder();
  const u16 = (n) => new Uint8Array([n & 255, (n >>> 8) & 255]);
  const u32 = (n) => new Uint8Array([n & 255, (n >>> 8) & 255, (n >>> 16) & 255, (n >>> 24) & 255]);
  const locals = [];
  const central = [];
  let offset = 0;
  for (const f of files) {
    const nameBytes = enc.encode(f.name);
    const crc = crc32(f.data);
    const size = f.data.length;
    const local = concatBytes([
      u32(0x04034b50), u16(20), u16(0), u16(0), u16(0), u16(0),
      u32(crc), u32(size), u32(size), u16(nameBytes.length), u16(0),
      nameBytes, f.data,
    ]);
    locals.push(local);
    central.push(concatBytes([
      u32(0x02014b50), u16(20), u16(20), u16(0), u16(0), u16(0), u16(0),
      u32(crc), u32(size), u32(size), u16(nameBytes.length),
      u16(0), u16(0), u16(0), u16(0), u32(0), u32(offset), nameBytes,
    ]));
    offset += local.length;
  }
  const centralBytes = concatBytes(central);
  const end = concatBytes([
    u32(0x06054b50), u16(0), u16(0), u16(files.length), u16(files.length),
    u32(centralBytes.length), u32(offset), u16(0),
  ]);
  return concatBytes([...locals, centralBytes, end]);
}
```

- [ ] **Step 2: Insert OOXML constants + cell helpers**

```js
const XLSX_CONTENT_TYPES = `<?xml version="1.0" encoding="UTF-8" standalone="yes"?>
<Types xmlns="http://schemas.openxmlformats.org/package/2006/content-types"><Default Extension="rels" ContentType="application/vnd.openxmlformats-package.relationships+xml"/><Default Extension="xml" ContentType="application/xml"/><Override PartName="/xl/workbook.xml" ContentType="application/vnd.openxmlformats-officedocument.spreadsheetml.sheet.main+xml"/><Override PartName="/xl/worksheets/sheet1.xml" ContentType="application/vnd.openxmlformats-officedocument.spreadsheetml.worksheet+xml"/><Override PartName="/xl/styles.xml" ContentType="application/vnd.openxmlformats-officedocument.spreadsheetml.styles+xml"/></Types>`;
const XLSX_RELS = `<?xml version="1.0" encoding="UTF-8" standalone="yes"?>
<Relationships xmlns="http://schemas.openxmlformats.org/package/2006/relationships"><Relationship Id="rId1" Type="http://schemas.openxmlformats.org/officeDocument/2006/relationships/officeDocument" Target="xl/workbook.xml"/></Relationships>`;
const XLSX_WORKBOOK = `<?xml version="1.0" encoding="UTF-8" standalone="yes"?>
<workbook xmlns="http://schemas.openxmlformats.org/spreadsheetml/2006/main" xmlns:r="http://schemas.openxmlformats.org/officeDocument/2006/relationships"><sheets><sheet name="Assets" sheetId="1" r:id="rId1"/></sheets></workbook>`;
const XLSX_WORKBOOK_RELS = `<?xml version="1.0" encoding="UTF-8" standalone="yes"?>
<Relationships xmlns="http://schemas.openxmlformats.org/package/2006/relationships"><Relationship Id="rId1" Type="http://schemas.openxmlformats.org/officeDocument/2006/relationships/worksheet" Target="worksheets/sheet1.xml"/><Relationship Id="rId2" Type="http://schemas.openxmlformats.org/officeDocument/2006/relationships/styles" Target="styles.xml"/></Relationships>`;
const XLSX_STYLES = `<?xml version="1.0" encoding="UTF-8" standalone="yes"?>
<styleSheet xmlns="http://schemas.openxmlformats.org/spreadsheetml/2006/main"><fonts count="2"><font><sz val="11"/><name val="Calibri"/></font><font><b/><sz val="11"/><name val="Calibri"/></font></fonts><fills count="7"><fill><patternFill patternType="none"/></fill><fill><patternFill patternType="gray125"/></fill><fill><patternFill patternType="solid"><fgColor rgb="FFD9D9D9"/></patternFill></fill><fill><patternFill patternType="solid"><fgColor rgb="FFA9D08E"/></patternFill></fill><fill><patternFill patternType="solid"><fgColor rgb="FFFFD966"/></patternFill></fill><fill><patternFill patternType="solid"><fgColor rgb="FFE06666"/></patternFill></fill><fill><patternFill patternType="solid"><fgColor rgb="FFF6B26B"/></patternFill></fill></fills><borders count="1"><border/></borders><cellStyleXfs count="1"><xf numFmtId="0" fontId="0" fillId="0" borderId="0"/></cellStyleXfs><cellXfs count="6"><xf numFmtId="0" fontId="0" fillId="0" borderId="0"/><xf numFmtId="0" fontId="1" fillId="2" borderId="0" applyFont="1" applyFill="1"/><xf numFmtId="0" fontId="0" fillId="3" borderId="0" applyFill="1"/><xf numFmtId="0" fontId="0" fillId="4" borderId="0" applyFill="1"/><xf numFmtId="0" fontId="0" fillId="5" borderId="0" applyFill="1"/><xf numFmtId="0" fontId="0" fillId="6" borderId="0" applyFill="1"/></cellXfs><cellStyles count="1"><cellStyle name="Normal" xfId="0" builtinId="0"/></cellStyles></styleSheet>`;

function xlsxEsc(s) {
  return String(s ?? "").replace(/&/g, "&amp;").replace(/</g, "&lt;").replace(/>/g, "&gt;").replace(/"/g, "&quot;").replace(/'/g, "&#39;");
}
function xlsxColLetter(i) {
  let s = "", n = i + 1;
  while (n > 0) { const m = (n - 1) % 26; s = String.fromCharCode(65 + m) + s; n = Math.floor((n - 1) / 26); }
  return s;
}
function xlsxCell(colIdx, rowNum, text, styleIdx) {
  const s = styleIdx ? ` s="${styleIdx}"` : "";
  return `<c r="${xlsxColLetter(colIdx)}${rowNum}"${s} t="inlineStr"><is><t xml:space="preserve">${xlsxEsc(text)}</t></is></c>`;
}
function assetRowStyle(r) {
  if (r._added) return 5;
  const s = r._status || "bad";
  if (s === "warn") return 4;
  if (s === "ok") return 2;
  return 3;
}
function assetStatusText(s) {
  if (s === "ok") return "Accounted For";
  if (s === "warn") return "Damaged";
  return "Missing";
}
```
(Note: helpers are named `xlsxEsc`/`xlsxColLetter` to avoid any clash with existing IM functions; `escapeHtml` already exists in IM and must not be shadowed.)

- [ ] **Step 3: Node validation of the writer**

Create `/tmp/.../scratchpad/im-xlsx-write-check.mjs`: paste copies of `crc32`+`CRC_TABLE`, `concatBytes`, `zipStore`, the `XLSX_*` constants, `xlsxEsc`, `xlsxColLetter`, `xlsxCell`, `assetRowStyle`, `assetStatusText`, plus a small builder that writes a 1-header + 2-row sheet (one `_status:"ok"`, one `_added:true`) and packages the 6 parts. Write to `im-test.xlsx`. Then: `unzip -l im-test.xlsx` lists all 6 parts, and (venv already at `/tmp/.../scratchpad/.venv`) `/tmp/.../scratchpad/.venv/bin/python -c "import openpyxl; wb=openpyxl.load_workbook('im-test.xlsx'); ws=wb.active; print([c.value for c in ws[1]]); print(ws['A2'].fill.fgColor.rgb)"` opens cleanly. Report the output. (Not committed.)

- [ ] **Step 4: Commit**

```bash
git add index.html
git commit -m "feat(export): port Inventory Check's .xlsx writer into IM"
```

---

### Task 7: Corrected Assets export → colored `.xlsx`; format-aware `exportCorrected`

**Files:** Modify `index.html:1251-1277` (`exportCorrected`), `index.html:909` (button label)

- [ ] **Step 1: Add `buildAssetsXlsx` and make `exportCorrected` format-aware**

Change `exportCorrected` — replace from `function exportCorrected() {` through its closing `}` (`index.html:1251-1277`). The current body is:
```js
function exportCorrected() {
  const kind = state.active;
  const tab = state.tabs[kind];
  const cols = tab.cols;
  const vocab = statusVocab(kind);
  const lines = [tab.headers.map(csvCell).join(",")];
  for (const r of tab.rows) {
    const cells = tab.headers.map(h => {
      if (cols.status && h === cols.status) return csvCell(vocab[r._status] ?? "");
      if (cols.note   && h === cols.note)   return csvCell(r._note);
      return csvCell(r[h]);
    });
    lines.push(cells.join(","));
  }
  const blob = new Blob([lines.join("\r\n")], { type: "text/csv" });
  const base = (tab.fileName || `${kind}.csv`).replace(/\.csv$/i, "");
  const a = document.createElement("a");
  a.href = URL.createObjectURL(blob);
  a.download = `${base}-corrected.csv`;
  document.body.appendChild(a);
  a.click();
  a.remove();
  setTimeout(() => URL.revokeObjectURL(a.href), 1000);
}
```
Replace it with:
```js
// Corrected Assets export: the new colored .xlsx (round-trips into Inventory Check).
function buildAssetsXlsx() {
  const tab = state.tabs.assets;
  const cols = tab.cols;
  const headers = ["Asset ID", "Cat Class", "S/N", "Make", "Model", "Market/Branch", "Status", "Added", "Notes"];
  const xmlRows = [`<row r="1">` + headers.map((h, i) => xlsxCell(i, 1, h, 1)).join("") + `</row>`];
  tab.rows.forEach((r, idx) => {
    const rowNum = idx + 2;
    const vals = [
      cols.id ? r[cols.id] : "", cols.cat ? r[cols.cat] : "", cols.ser ? r[cols.ser] : "",
      cols.make ? r[cols.make] : "", cols.model ? r[cols.model] : "", cols.market ? r[cols.market] : "",
      assetStatusText(r._status), r._added ? "Yes" : "", r._note || "",
    ];
    const sIdx = assetRowStyle(r);
    xmlRows.push(`<row r="${rowNum}">` + vals.map((v, i) => xlsxCell(i, rowNum, String(v ?? ""), sIdx)).join("") + `</row>`);
  });
  const sheet = `<?xml version="1.0" encoding="UTF-8" standalone="yes"?>
<worksheet xmlns="http://schemas.openxmlformats.org/spreadsheetml/2006/main"><sheetData>${xmlRows.join("")}</sheetData></worksheet>`;
  const enc = (s) => new TextEncoder().encode(s);
  const files = [
    { name: "[Content_Types].xml", data: enc(XLSX_CONTENT_TYPES) },
    { name: "_rels/.rels", data: enc(XLSX_RELS) },
    { name: "xl/workbook.xml", data: enc(XLSX_WORKBOOK) },
    { name: "xl/_rels/workbook.xml.rels", data: enc(XLSX_WORKBOOK_RELS) },
    { name: "xl/styles.xml", data: enc(XLSX_STYLES) },
    { name: "xl/worksheets/sheet1.xml", data: enc(sheet) },
  ];
  return new Blob([zipStore(files)], { type: "application/vnd.openxmlformats-officedocument.spreadsheetml.sheet" });
}
function exportCorrected() {
  const kind = state.active;
  const tab = state.tabs[kind];
  const base = (tab.fileName || `${kind}`).replace(/\.(csv|xlsx)$/i, "");
  let blob, ext;
  if (kind === "assets") {
    blob = buildAssetsXlsx();
    ext = "xlsx";
  } else {
    const cols = tab.cols;
    const vocab = statusVocab(kind);
    const lines = [tab.headers.map(csvCell).join(",")];
    for (const r of tab.rows) {
      const cells = tab.headers.map(h => {
        if (cols.status && h === cols.status) return csvCell(vocab[r._status] ?? "");
        if (cols.note   && h === cols.note)   return csvCell(r._note);
        return csvCell(r[h]);
      });
      lines.push(cells.join(","));
    }
    blob = new Blob([lines.join("\r\n")], { type: "text/csv" });
    ext = "csv";
  }
  const a = document.createElement("a");
  a.href = URL.createObjectURL(blob);
  a.download = `${base}-corrected.${ext}`;
  document.body.appendChild(a);
  a.click();
  a.remove();
  setTimeout(() => URL.revokeObjectURL(a.href), 1000);
}
```

- [ ] **Step 2: Relabel the export button**

Change `index.html:909` from:
```html
        <button id="exportBtn" class="btn-ghost" style="padding:4px 10px;min-height:0">Export Corrected CSV</button>
```
to:
```html
        <button id="exportBtn" class="btn-ghost" style="padding:4px 10px;min-height:0">Export Corrected</button>
```

- [ ] **Step 3: Verify (browser)**

Load an Assets `.xlsx`, fix a Missing asset to Accounted (via the exceptions editor), click **Export Corrected** → downloads `…-corrected.xlsx`. Open in Google Sheets: 9 columns, the fixed row is green + Status `Accounted For`. Re-import that file into **Inventory Check** → loads as assets with the corrected status. Switch to a Parts tab → **Export Corrected** still downloads `…-corrected.csv`.

- [ ] **Step 4: Commit**

```bash
git add index.html
git commit -m "feat(export): assets corrected export as colored .xlsx; parts/bulk stay CSV"
```

---

### Task 8: Full QA pass + tag v0.7.0

**Files:** none (verification + tag)

- [ ] **Step 1: End-to-end manual QA** (mirrors spec "Testing")

1. IC v1.7.0 Assets `.xlsx` → loads; stats show Accounted/Missing/Damaged; completeness = accounted / total; Missing under Missing exceptions.
2. Old Assets CSV (Check Status headers) → still imports.
3. Parts/Bulk CSV → unchanged (Match/Short/Over, Not Counted).
4. Fix a Missing → Accounted; Export Corrected → `…-corrected.xlsx` opens in Sheets (colors + `Accounted For`), re-imports into Inventory Check.
5. Re-save the IC `.xlsx` in Excel, load into IM → clear "export a fresh file" message, no crash.

- [ ] **Step 2: JS syntax gate**

```bash
cd /home/cj/development/inventory-manager
python3 -c "import re; h=open('index.html').read(); s=re.findall(r'<script[^>]*>(.*?)</script>', h, re.S); open('/tmp/im_bundle.js','w').write('\n;\n'.join(s))"
node --check /tmp/im_bundle.js && echo SYNTAX_OK
```
Expected: `SYNTAX_OK`.

- [ ] **Step 3: Tag**

```bash
git tag v0.7.0
```

---

## Self-Review

**Spec coverage:**
- Part A (dispatch + ingestGrid + accept): Task 4. ✓
- Part B (`parseXlsxGrid` + resave guard): Task 3. ✓
- Part C (aliases + decodeStatus + ser): Task 2. ✓
- Part D (completeness): Task 5. ✓ (exceptions/breakdown fall out of Missing→`bad`, no code change — noted in spec.)
- Part E (writer port + buildAssetsXlsx + exportCorrected + button): Tasks 6, 7. ✓
- Part F (version + README): Task 1; tag in Task 8. ✓
- Testing: Task 8. ✓

**Placeholder scan:** No TBD/TODO; every code step shows complete code. ✓

**Type/name consistency:** `parseXlsxGrid`/`xlsxReadParts`/`xmlUnescape`/`colIndexFromRef` (reader); `crc32`/`concatBytes`/`zipStore`/`XLSX_*`/`xlsxEsc`/`xlsxColLetter`/`xlsxCell`/`assetRowStyle`/`assetStatusText`/`buildAssetsXlsx` (writer) — used consistently. Reader uses `xmlUnescape`; writer uses `xlsxEsc` (distinct, no clash). Writer helpers deliberately prefixed `xlsx*` to avoid clashing with IM's existing `escapeHtml`. Style indices (1=header,2=green,3=yellow,4=red,5=orange) match `assetRowStyle` and `XLSX_STYLES` `cellXfs`. `ingestGrid` defined in Task 4 and called by both the CSV wrapper and the xlsx path. ✓
