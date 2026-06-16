# CV Accuracy Dashboard — Handoff Document

---

## Section 1: Quick Start

### What It Does
A single-file browser app (`index.html`) for analyzing computer vision model accuracy on NARTD cooler shelf images for Coca-Cola India (CCI). CV/QC teams upload three files — bounding-box detection data, image quality mapping, and product masterdata — and get back accuracy KPIs, charts, drill-downs, and per-image breakdown tables. Zero server involvement; all computation runs client-side.

### Hosted URL
**https://kartik-15.github.io/cv-accuracy-dashboard/**

GitHub repository: `https://github.com/Kartik-15/cv-accuracy-dashboard`
Deployment: GitHub Pages, served from `main` branch root. `.nojekyll` file prevents Jekyll from intercepting the single HTML file.

### File/Module Status

| File | Status | Note |
|------|--------|------|
| `index.html` | **ACTIVE** | Entire app — HTML + CSS + JS in one file (~2200+ lines) |
| `.nojekyll` | **ACTIVE** | Required for GitHub Pages to serve `index.html` directly |
| `CGC_New_15th June.csv` | **ACTIVE** | Canonical masterdata sample (CGC pivot format) |
| `Accuracy Sheet.xlsx` | **ACTIVE** | Sample detection data (use `Raw` sheet) |
| `CCI_Image_Quality_Analysis - 12th June - Sheet1.csv` | **ACTIVE** | Sample image quality file |
| `CCI_Rollout_CGC_15th June.csv` | **SUPERSEDED** | Older masterdata; replaced by `CGC_New` |
| `Final Brand and Variant.csv` | **SUPERSEDED** | Pre-flattened masterdata; app now builds this map internally |
| `RA Accuracy.xlsx` | **UTILITY** | Earlier accuracy sheet; same schema |

### Current Objective
Core accuracy analysis + trend tracking is feature-complete. The next major feature is **MSL (Must-Stock List)** — store-level compliance columns added to the Size Demo table, requiring a 4th file upload.

### Immediate Next Steps
1. Implement MSL feature: add a 4th upload zone for store-level MSL file, join on store/session ID, append MSL compliance columns to the Size Demo table.
2. Add export for Brand Accuracy and Variant Accuracy tables (currently only Brand Demo and Size Demo export).
3. Test with larger datasets to identify in-browser performance limits.

### Files Most Likely to Be Edited Next

| Path | Why |
|------|-----|
| `index.html` upload screen | Add 4th zone for MSL file |
| `index.html` `APP` state | Add `msl` key to `files`, `raw`, `cfg` |
| `index.html` `buildSizeDemoRows` / `renderSizeDemo` | Inject MSL compliance columns |

---

## Section 2: Project Overview

### Full Capabilities (current state)

1. **Three-file upload** — Detection Data (XLSX/CSV), Image Quality (XLSX/CSV), Product Masterdata (XLSX/CSV). Drag-and-drop or click-to-browse, with staged file confirmation.
2. **Model ID + Date inputs** — Entered before processing; stored per-run for trend tracking. Date defaults to today on page load.
3. **Flexible column mapping modal** — auto-detects column names via candidate list matching; users can override via dropdown before processing. Handles XLSX sheet selection.
4. **Quality normalization** — maps varied strings ("Good Quality", "Blurred image", "blank image") to canonical tiers: Good / Average / Poor / Blank / Blurred / Impossible / Unknown.
5. **Masterdata pivot parsing** — converts CGC-format rows (`class_name`, `attribute_name`, `attribute_value`) into a flat `displayName → {brand, variant, measure, subcat, competition}` lookup map.
6. **Enriched bounding-box rows** — joins quality and masterdata attributes onto each detection row; computes `is_correct` fresh from GT vs Prediction string equality.
7. **Headline stat cards** — Images, Unique Brands, Unique Variants, Unique Groups (subcats). Always visible at top of dashboard, filter-reactive.
8. **KPI cards** — Object Accuracy, Macro Precision, Macro Recall, Macro F1 with color-coded threshold badges (≥90% emerald, 75–89% amber, <75% red).
9. **Multi-tab layout** — 7 main tabs: Overview, Brand Analysis, Variant Analysis, SKU Analysis, Brand Demo, Size Demo, Trends.
10. **Lazy tab rendering** — only the active tab's charts/tables re-render on filter change, avoiding hidden-canvas sizing bugs.
11. **Overview tab** — Accuracy by Image Quality chart (c1) + Brand/Variant/SKU top-10 grouped bar (c2).
12. **Brand Analysis tab** — Top-15 horizontal bar chart (c3) + brand accuracy pivot with click-to-expand drill-down. Drill-down shows predicted brand names (aggregated), error counts, % of errors, and up to 6 inline image chip links per brand.
13. **Variant Analysis tab** — Top-15 horizontal bar chart (c4) + variant accuracy pivot with same drill-down structure as Brand.
14. **SKU Analysis tab** — SKU accuracy table sorted by error count descending, with click-to-expand misclassification drill-down (same brand-level aggregation + image chips).
15. **Brand Demo tab** — Image-level aggregation for CCI (self) GT items only. Top-15 competitor brands by actual error frequency (dynamic from filtered data), not all masterdata brands. Paginated (25/page), searchable, CSV exportable.
16. **Size Demo tab** — Image-level aggregation for same-brand wrong-size errors. Paginated, searchable, CSV exportable.
17. **Trends tab** — Historical KPI line chart (c5), Brand accuracy trend selector + chart (c6), Variant accuracy trend selector + chart (c7), SKU accuracy trend selector + chart (c8). History table with per-run delete + Clear All. Data persisted in localStorage under key `cci_accuracy_history_v1`.
18. **Sidebar filters** — Image Quality, GT Brand, Variant/Pack Type, SKU Size (ml). All filters apply across all tabs simultaneously via `filteredRows()`. Reset All button.
19. **Image viewer links** — session IDs in drill-downs rendered as `.img-chip` elements linking to `https://view.shelfwatch.io?url={file_path}` with quality badge overlay.

### Current Status
**Deployed** at https://kartik-15.github.io/cv-accuracy-dashboard/ — shareable with the CCI team. All core features including trend tracking are live.

### Key Constraints and Non-Obvious Rules
- `is_correct` is **always recomputed** as `QC_Display Name === Display Name` string equality. The `DN_Acc` column in the XLSX is deliberately ignored — it was found to be unreliable.
- The `competition` column in the Raw detection sheet reflects the **prediction's** competition type, not the GT item's. GT competition must be looked up from the masterdata map.
- Brand Demo only includes bounding boxes where the **GT item is 'self'** (CCI brand).
- In Brand Demo, items with an unknown `pred_brand` fall into the **Others** bucket.
- Size Demo only captures errors where **GT brand === Pred brand** AND **GT measure ≠ Pred measure** AND both measures are non-zero.
- Masterdata key lookup is always **case-insensitive** (`displayName.toLowerCase()`).
- Brands excluded from the unique-brand column list: empty string, null, `'Others'`, `'Non-NARTD'`, `'#N/A'`.
- Quality fallback order: IQ file lookup → embedded `quality` column in detection file → `'Unknown'`.
- CSV export prepends a UTF-8 BOM (`﻿`) so Excel opens it without encoding issues.
- The Detection Data XLSX defaults to the `Raw` sheet if it exists; otherwise falls back to the first sheet.
- Trend data persists across page loads in localStorage but is cleared if the user manually clears browser storage.

---

## Section 3: Core Logic / Algorithm

### is_correct Computation
```js
const is_correct = gt !== '' && gt === pred;
// gt  = cl(row[cols.qc_dn])   — QC Display Name (Ground Truth)
// pred = cl(row[cols.dn])     — Display Name (AI Prediction)
// cl() = v => String(v).trim()
```

### Masterdata Map Build (`buildMasterdataMap`)
```js
// Input: CGC pivot rows [{class_name, attribute_name, attribute_value, competition}]
// Output: { "display name lowercase": {brand, variant, measure, subcat, competition} }

for each row:
  group by class_name → collect attribute_name:attribute_value pairs
  look up "Display Name" (or "display_name") attribute → use as the map key
  extract: Brand, Pack type (→ variant), measure (→ int ml), Sub category, competition
```

### Quality Normalization (`normalizeQuality`)
```
Input string (lowercased) → Canonical output
starts with "good"                          → "Good"
starts with "average" OR includes "avg"     → "Average"
starts with "poor"                          → "Poor"
includes "blank" OR "empty"                 → "Blank"
includes "blurred" OR "blur"                → "Blurred"
starts with "impossible"                    → "Impossible"
anything else                               → pass through as-is
```

### Metrics Computation (`computeMetrics`)
```
For each bounding-box row:
  - Increment stats[gt].gtN
  - Increment stats[pred].predN
  - If is_correct: increment stats[gt].TP and correct count

Macro Precision = mean(TP/predN) for all classes with predN > 0
Macro Recall    = mean(TP/gtN)   for all classes with gtN > 0
Macro F1        = 2 * P * R / (P + R)
Object Accuracy = correct / total_rows
```

### Brand Demo Error Taxonomy (GT 'self' items only, mutually exclusive per bbox)

| Priority | Category | Condition |
|----------|----------|-----------|
| 1 | Self Facings (Correct) | `is_correct === true` |
| 2 | Self Misclassification | `!is_correct && gt_brand === pred_brand` |
| 3 | Non-NARTD | `pred === 'Non-NARTD'` |
| 4 | Others | `pred === 'Others'` OR pred_brand not in brand column list |
| 5 | [Brand X] | `!is_correct && gt_brand !== pred_brand && pred_brand is a known brand` |

### Drill-Down Error Aggregation (Brand, Variant, SKU tabs)
```js
// getPredBrandLabel(r) — maps a prediction row to its brand name
// 'Others' and 'Non-NARTD' pass through; all others → r.pred_brand

// buildBrandAccuracy / buildVariantAccuracy / buildSkuAccuracy
// errors structure per pivot row:
//   { brandName: { count: number, imgs: { sid: { sid, fp, quality } } } }
// Key = getPredBrandLabel(r) — brand-level, not Display Name

// buildDrillContent(errors, total, colspan)  — 3 args
// Renders table: Predicted Brand | Count | % of Errors | Affected Images
// Up to 6 img chips per brand row (via buildImgChips)
```

### Top Competitor Brands for Brand Demo (`getTopCompetitorBrandsForDemo`)
```js
// Counts competitor brand predictions for self-GT error rows in filtered data
// Returns top N (default 15) by frequency
// Used dynamically so Brand Demo columns adapt to filtered data
```

### Trend Tracking
```js
HISTORY_KEY = 'cci_accuracy_history_v1'

// saveSnapshot() — called automatically after processData() completes
// Stores in localStorage: { [modelId__date]: { kpis, brandStats, variantStats, skuStats(top150) } }

// Each snapshot key: `${modelId}__${date}` (double underscore separator)
// History table shows all snapshots; user can delete individual runs or Clear All
// Charts c5–c8 use Chart.js line charts with spanGaps:true for missing data points
// TREND_COLORS = 12-color array cycling for multi-series charts
```

### Processing Prerequisites (ordered)
1. All three files staged + model ID entered + date selected → **Process button enabled**
2. Column mapping confirmed, all required fields mapped → **Processing starts**
3. `buildMasterdataMap` completes → `APP.d.mMap` populated
4. `buildQualityMap` completes → `APP.d.qMap` populated
5. `enrichRows` completes → `APP.d.rows` populated
6. Unique filter values extracted
7. `renderAll()` called → `saveSnapshot()` called after render

**Stop conditions:** Missing required column mapping → alert, abort. File parse error → alert, abort.

---

## Section 4: Architecture

### End-to-End Flow

```
┌──────────────────────────────────────────────────────┐
│                   UPLOAD SCREEN                      │
│  [Detection XLSX/CSV] [Image Quality CSV]            │
│  [Masterdata CSV]                                    │
│  Model ID: [text input]   Date: [date picker]        │
│  [Process Data button]                               │
└──────────────┬───────────────────────────────────────┘
               │ openMappingModal()
               ▼
┌──────────────────────────────────────────────────────┐
│               COLUMN MAPPING MODAL                   │
│  parseFileAuto() × 3 files                           │
│  → auto-detect columns via bestMatch()               │
│  User confirms / overrides                           │
└──────────────┬───────────────────────────────────────┘
               │ confirmProcess()
               ▼
┌──────────────────────────────────────────────────────┐
│               PROCESSING PIPELINE                    │
│  buildMasterdataMap(md rows, cols) → APP.d.mMap      │
│  buildQualityMap(iq rows, cols)    → APP.d.qMap      │
│  enrichRows(det rows, qMap, mMap)  → APP.d.rows      │
│  extract unique filter values                        │
│  renderAll() → saveSnapshot()                        │
└──────────────┬───────────────────────────────────────┘
               │
               ▼
┌──────────────────────────────────────────────────────┐
│                   DASHBOARD                          │
│                                                      │
│  filteredRows()  ← sidebar filter state (APP.f)     │
│       │                                              │
│       ├─ renderHeadlineStats() → 4 stat cards        │
│       ├─ renderKPIs()          → 4 KPI cards         │
│       ├─ renderFilters()       → sidebar update      │
│       └─ renderActiveTabContent(rows)                │
│             │                                        │
│             ├─ overview:   c1 (quality bar), c2 (brand bar)
│             ├─ brand:      c3 (top-15 bar) + pivot + drill-downs
│             ├─ variant:    c4 (top-15 bar) + pivot + drill-downs
│             ├─ sku:        SKU accuracy table + drill-downs
│             ├─ brandDemo:  dynamic-column table + CSV export
│             ├─ sizeDemo:   size error table + CSV export
│             └─ trends:     c5 (KPI), c6 (brand), c7 (variant), c8 (SKU)
│                            + history table (delete / clear all)
└──────────────────────────────────────────────────────┘
```

### Core Data Structures

```js
// Enriched bounding-box row (APP.d.rows[i])
{
  sid:          string,   // session_id — unique image identifier
  gt:           string,   // QC Display Name (Ground Truth label)
  pred:         string,   // Display Name (AI Prediction label)
  comp:         string,   // 'self'|'competitor' — PREDICTION's competition (from raw data)
  fp:           string,   // GCS file path (wrap with https://view.shelfwatch.io?url=)
  quality:      string,   // Normalized: 'Good'|'Average'|'Poor'|'Blank'|'Blurred'|'Impossible'|'Unknown'
  is_correct:   boolean,  // gt === pred (recomputed; never read from file)
  gt_brand:     string,   // Brand of GT item (from masterdata)
  gt_variant:   string,   // Pack type of GT item (from masterdata)
  gt_measure:   number,   // Size in ml of GT item (int, 0 if unknown)
  gt_comp:      string,   // 'self'|'competitor' — GT item's competition (from masterdata)
  pred_brand:   string,   // Brand of predicted item (from masterdata)
  pred_variant: string,   // Pack type of predicted item
  pred_measure: number    // Size in ml of predicted item
}

// Masterdata map (APP.d.mMap)
{ "display name lowercase": { brand, variant, measure, subcat, competition } }

// Quality map (APP.d.qMap)
Map<sessionId: string, quality: string>

// Error structure in buildBrandAccuracy / buildVariantAccuracy / buildSkuAccuracy
// errors per pivot row:
{ brandName: { count: number, imgs: { sid: { sid, fp, quality } } } }

// localStorage snapshot (per model run)
{
  modelId: string,
  date:    string,
  kpis:    { acc, precision, recall, f1 },
  brandStats:   [ { brand, acc, correct, total } ],
  variantStats: [ { variant, acc, correct, total } ],
  skuStats:     [ { sku, acc, correct, total } ]   // top 150 only
}

// APP state (complete)
APP = {
  files: { det, iq, md },             // raw File objects
  raw:   { det, iq, md },             // parsed: { isXLSX, sheets, activeSheet, data }
  cfg:   { det, iq, md },             // { sheet: string, cols: { key: columnName } }
  d: {
    rows, qMap, mMap,
    brands, qualities, variants, measures, subcats,
    modelId: string,
    date: string
  },
  f:     { qualities:[], brands:[], variants:[], measures:[] },
  pg:    { bd: {page,size,q,rows}, sd: {page,size,q,rows} },
  tabs:  { main: 'overview' },
  charts: { c1, c2, c3, c4, c5, c6, c7, c8 },
  _sdBrands: [],           // brands with size errors (dynamic, side-effect of buildSizeDemoRows)
  _skuExpanded: Set,       // expanded SKU drill-down rows
  _skuQ: string,           // SKU search query
  _pivotSku: [],           // current SKU pivot data
  _trendBrands: [],        // selected brand items for trend chart
  _trendVariants: [],      // selected variant items for trend chart
  _trendSkus: [],          // selected SKU items for trend chart
  _brandExpanded: Set,     // expanded brand drill-down rows
  _variantExpanded: Set,   // expanded variant drill-down rows
  _pivotBrand: [],         // current brand pivot data
  _pivotVariant: []        // current variant pivot data
}
```

### File/Module Map

```
Accuracy Dashboard/
├── index.html                    ACTIVE — entire app (HTML + CSS + JS, ~2200+ lines)
│   ├── CSS (.main-tab, .main-tab.active, .drill-wrap, .img-chip, .hstat* classes added)
│   ├── Upload Screen (#uploadScreen)
│   │   └── Model ID input + Date picker (before Process button)
│   ├── Column Mapping Modal (#mappingModal)
│   ├── Processing Overlay (#procOverlay)
│   ├── Dashboard (#dashboard)
│   │   ├── Top bar (model ID + date + images + quality-matched stats, re-upload)
│   │   ├── Headline stat cards (Images, Brands, Variants, Groups)
│   │   ├── Main tab nav (Overview | Brand | Variant | SKU | Brand Demo | Size Demo | Trends)
│   │   ├── Sidebar (filter sections)
│   │   └── Tab content divs (one per main tab, hidden/shown)
│   └── JavaScript
│       ├── APP state object
│       ├── HISTORY_KEY + localStorage trend persistence
│       ├── File staging & drag-drop
│       ├── parseXLSX / parseCSV / parseFileAuto
│       ├── normalizeQuality
│       ├── COL_DEFS + bestMatch (column auto-detection)
│       ├── buildTabHTML / switchTab / openMappingModal
│       ├── processData (pipeline orchestrator; reads modelId/date; calls saveSnapshot)
│       ├── buildMasterdataMap / buildQualityMap / enrichRows
│       ├── filteredRows / renderFilters / toggleFilter
│       ├── computeMetrics
│       ├── renderHeadlineStats / renderKPIs
│       ├── MAIN_TABS / switchMainTab / renderActiveTabContent
│       ├── renderChart1 (quality bar, c1) / renderChart2 (brand bar, c2)
│       ├── getPredBrandLabel (maps pred row → brand label)
│       ├── buildBrandAccuracy / renderBrandAccuracy / renderChart3 (c3)
│       ├── buildVariantAccuracy / renderVariantAccuracy / renderChart4 (c4)
│       ├── buildSkuAccuracy / renderSkuAccuracy / toggleSkuDrillIdx
│       ├── buildDrillContent (errors, total, colspan) / buildImgChips
│       ├── getTopCompetitorBrandsForDemo / buildBrandDemoRows / renderBrandDemo
│       ├── buildSizeDemoRows / renderSizeDemo
│       ├── renderPag / changePg / exportCSV
│       ├── saveSnapshot / deleteSnapshot / clearHistory
│       ├── renderTrendsTab / renderHistoryTable
│       ├── renderKpiTrendChart (c5)
│       ├── renderBrandTrendSelector / renderBrandTrendChart (c6)
│       ├── renderVariantTrendSelector / renderVariantTrendChart (c7)
│       ├── renderSkuTrendSelector / renderSkuTrendChart (c8)
│       ├── toggleTrendItem(type, item)
│       └── resetAll (destroys c1–c8, resets all trend/expand state)
├── .nojekyll                     ACTIVE — GitHub Pages bypass (empty file)
├── Accuracy Sheet.xlsx           Sample detection data (use "Raw" sheet)
├── CCI_Image_Quality_Analysis - 12th June - Sheet1.csv   Sample quality file
├── CGC_New_15th June.csv         CANONICAL sample masterdata
├── CCI_Rollout_CGC_15th June.csv Older masterdata (superseded)
├── Final Brand and Variant.csv   Pre-flattened masterdata (superseded)
└── RA Accuracy.xlsx              Earlier accuracy sheet (utility reference)
```

---

## Section 5: Usage & Output

### Running the App
```
# Hosted (share with team):
https://kartik-15.github.io/cv-accuracy-dashboard/

# Local development:
open "/Users/kartik/Desktop/Accuracy Dashboard/index.html"
# Works in Chrome, Safari, Firefox (Chrome recommended).
```

### Deploying Changes
```bash
cd "/Users/kartik/Desktop/Accuracy Dashboard"
git add index.html
git commit -m "describe change"
git push origin main
# GitHub Pages auto-deploys within ~30 seconds.
```

### File Upload Requirements

| Slot | Format | Required Columns |
|------|--------|-----------------|
| Detection Data | XLSX (use `Raw` sheet) or CSV | `session_id`, `QC_Display Name`, `Display Name`, `competition` |
| Image Quality | CSV | `cooler_session_id` (or `session_id`), `image quality` (or `quality`) |
| Masterdata | CSV (CGC pivot) | `class_name`, `attribute_name`, `attribute_value` |

Optional detection columns: `file_path` (for image viewer chips), `quality` (embedded quality fallback).

Before clicking Process, enter:
- **Model ID** — free text (e.g. `v2.3`, `iteration-15`). Used as part of the trend history key.
- **Date** — date picker, defaults to today.

### KPI Output Schema

| Metric | Formula | Color Threshold |
|--------|---------|----------------|
| Object Accuracy | correct / total | ≥90% green, 75–89% amber, <75% red |
| Macro Precision | mean(TP/predN per class) | same |
| Macro Recall | mean(TP/gtN per class) | same |
| Macro F1 | 2·P·R / (P+R) | same |

*(Total Bounding Boxes KPI card was removed; bbox count moved to the top-bar stats area.)*

### Headline Stats (always visible, filter-reactive)

| Card | Value |
|------|-------|
| Images | Distinct `session_id` values in filtered rows |
| Unique Brands | Distinct `gt_brand` values |
| Unique Variants | Distinct `gt_variant` values |
| Unique Groups | Distinct `gt_subcat` values |

### Drill-Down Output (Brand / Variant / SKU tabs)
Each row in the accuracy pivot table is clickable. Expanding shows:

| Column | Content |
|--------|---------|
| Predicted Brand | Aggregated brand name (not Display Name / SKU name) |
| Count | Number of bounding boxes misclassified as this brand |
| % of Errors | Share of that item's total errors |
| Affected Images | Up to 6 clickable `.img-chip` elements with session ID + quality badge |

### Brand Demo Export (`brand_demo_sheet.csv`)

| Column | Definition |
|--------|-----------|
| Image ID | `session_id` |
| Image Quality | Normalized quality tier |
| Self Facings | Correct CCI predictions in this image |
| Self Misclassification | CCI GT misclassified as same brand, different SKU |
| Non-NARTD | Predicted as `Non-NARTD` |
| Others | Predicted as `Others` OR unknown brand |
| [Brand X] × top 15 | Count predicted as this competitor brand (dynamic columns) |
| Total Self GT | Total CCI GT bounding boxes in this image |

### Size Demo Export (`size_demo_sheet.csv`)

| Column | Definition |
|--------|-----------|
| Image ID | `session_id` |
| Image Quality | Normalized quality tier |
| Total Size Errors | Same-brand, different-measure errors in this image |
| [Brand X] | Size errors attributable to this brand |

### Trend Tracking
Each time "Process Data" completes, a snapshot is saved to localStorage. In the **Trends tab**:
- **History table** — one row per model run (model ID, date, Object Accuracy, F1). Delete individual runs or Clear All.
- **KPI Trend chart (c5)** — line chart of Accuracy, Precision, Recall, F1 across all saved runs.
- **Brand / Variant / SKU charts (c6–c8)** — multi-select pickers to choose items; line charts show accuracy over model cycles.

---

## Section 6: Decisions Log

**1. Recompute `is_correct` from GT vs Pred strings, never read `DN_Acc`**
- `is_correct = (QC_Display Name === Display Name)` in `enrichRows`. The `DN_Acc` column was found unreliable.

**2. GT competition from masterdata, not from the `competition` column in Raw**
- `competition` in Raw reflects the prediction's product type. GT competition uses `mMap[gt.toLowerCase()].competition`.

**3. Brand Demo includes only GT 'self' items**
- The Brand Demo sheet is CCI-internal. Competitor GT items are out of scope.

**4. Unrecognized pred_brand falls into Others in Brand Demo**
- Edge case where masterdata doesn't cover a prediction label — lumping into Others is safer than creating phantom columns.

**5. Drill-downs aggregate by brand name, not Display Name**
- **Decision:** `getPredBrandLabel(r)` maps any prediction to its brand; error accumulator uses brand as key.
- **Reasoning:** Users want to see "was Thums UP mistaken as Pepsi?" not a list of 40 individual SKUs for Pepsi. Brand-level aggregation is more actionable.
- **Effect:** `buildDrillContent` signature changed from 4 args to 3 — `(errors, total, colspan)`. The `errorImgs` separate arg was eliminated; imgs are now embedded inside each brand key.

**6. Brand Demo uses top-15 competitor brands by actual error frequency, not all masterdata brands**
- **Decision:** `getTopCompetitorBrandsForDemo(rows, 15)` counts competitor brand predictions in filtered data; top 15 become columns.
- **Reasoning:** Masterdata can have 50+ brands; showing all makes the table unusable. Dynamic top-15 keeps columns meaningful and filter-reactive.

**7. Lazy tab rendering via `renderActiveTabContent()`**
- **Decision:** Each tab's charts and tables only render when that tab is active. `switchMainTab()` calls `renderActiveTabContent(filteredRows())`.
- **Reasoning:** Chart.js cannot measure canvas dimensions when the canvas is hidden (`display:none`). Rendering only on activation ensures correct sizing.

**8. Trend snapshots keyed by `${modelId}__${date}`**
- Double-underscore separator chosen to be unlikely to appear in model IDs or dates naturally. Snapshots are overwritten if the same model ID + date is processed again.

**9. Trend snapshot stores only top-150 SKUs**
- Full SKU list can be thousands of entries. localStorage has a 5–10 MB limit. Top-150 by error count covers the useful analysis range.

**10. MSL feature deferred**
- Requires a 4th file upload with store-level MSL data; out of scope for current version.

**11. CGC pivot format as canonical masterdata**
- `CGC_New_15th June.csv` (pivot format) is canonical. `Final Brand and Variant.csv` (pre-flattened) is superseded.

---

## Section 7: Bugs & Corrections

**1. `competition` column used for GT classification instead of masterdata lookup**
- Fixed in initial development: `gt_comp` now comes from `mMap[gt.toLowerCase()].competition`.

**2. Quality normalization didn't handle variant strings from actual data**
- Fixed: `normalizeQuality()` added with prefix/substring matching for "Good Quality", "Blurred image", etc.

**3. Masterdata lookup key case mismatch**
- Fixed: all masterdata map keys stored and looked up as `.toLowerCase()`.

**4. Brand Demo "Others" bucket swallowing valid named-brand errors**
- Fixed: check `pred_brand in img.bc` before bucketing into Others.

**5. `buildDrillContent` signature mismatch after error structure refactor**
- **Problem:** After changing the error accumulator to embed imgs inside each brand key and dropping the separate `errorImgs` arg, all 3 callers (brand, variant, SKU) still passed 4 arguments.
- **Fix:** Updated all 3 callers to `buildDrillContent(row.errors, row.errorCount, colspan)`.
- **When fixed:** During the drill-down + image chips improvement session.

**6. Chart.js canvas sizing in hidden tabs**
- **Problem:** Charts rendered into hidden `<div>` (display:none) tabs showed as 0×0 or wrong size.
- **Fix:** Lazy rendering — tabs only render their charts when `switchMainTab()` makes them active.

---

## Section 8: Open Tasks

### Critical
- None blocking current use.

### Important
- **MSL feature:** Add 4th upload zone + MSL file parsing + join on store/session + MSL compliance columns in Size Demo table.
- **Export for Brand Accuracy and Variant Accuracy tables:** Currently only Brand Demo and Size Demo have CSV export.

### Nice-to-Have
- Performance testing with large datasets (10k+ bounding boxes).
- Add a Confusion Matrix view (top GT → top predicted).
- Add session-level filter (filter by specific image session IDs).
- Persist last-used column mapping preferences (localStorage).
- Add a "Download All" button that exports all tables in one zip.
- Show more than 6 image chips per brand in drill-downs (currently capped for layout).

### Explicitly Out of Scope (confirmed)
- Server-side processing or API backend — by design, all computation stays client-side.
- Multi-user auth or session management.
- MSL columns (deferred to later version).

---

## Section 9: Known Limitations

**1. Trend data stored in browser localStorage**
- If a user clears browser storage or switches browsers/machines, trend history is lost. Not synced between users.

**2. All bounding boxes weighted equally in Object Accuracy**
- Object Accuracy is `correct / total` across all bboxes, regardless of image quality or GT brand.

**3. Macro metrics include 'Others' and 'Non-NARTD' as classes**
- These labels appear as GT values in some rows and are included in Macro P/R/F1 computation. Intentional.

**4. Size Demo brand columns are dynamic, not fixed**
- Brand columns built from whichever brands have size errors in the filtered data. Filter-reactive by design.

**5. Session ID join is exact-match only**
- If `session_id` in detection data and `cooler_session_id` in image quality file differ by any character, the join fails silently. The quality join warning banner catches total-miss cases but not partial misses.

**6. Image viewer requires GCS URL format**
- The `.img-chip` viewer link only works if `file_path` is a valid GCS URL recognizable by view.shelfwatch.io.

**7. Top-150 SKU cap in trend snapshots**
- Trend charts for SKU accuracy only cover the top 150 SKUs by error count at snapshot time.

---

## Section 10: Preferences & Conventions

- **Single-file architecture is intentional.** Distributed by sharing `index.html` or the GitHub Pages URL. No build tools, no npm, no bundling. Libraries loaded from CDN (Tailwind, FontAwesome, PapaParse, Chart.js, SheetJS).
- **Dark theme is fixed** (`--bg: #0a0f1e`). No light mode planned.
- **Table columns: right-align numbers, left-align labels** — enforced via `.left` class. Don't deviate.
- **KPI color thresholds are hardcoded**: ≥90% = green, 75–89% = amber, <75% = red. Business-defined, not configurable.
- **`cl()` is the universal cell normalizer** — always use `cl(row[colName])` when reading detection data. It handles null/undefined/whitespace.
- **Chart.js instances c1–c8 are destroyed and recreated** each time their tab activates. `APP.charts.cN.destroy()` before creating new. Don't skip this or you'll leak canvas contexts.
- **`resetAll()` destroys c1–c8** and resets all expand sets and trend selection arrays when re-uploading.
- **Pagination state resets to 1** whenever filters change or new data is processed.
- **`APP._sdBrands` is a side-effect of `buildSizeDemoRows`** — set inside that function; always call `buildSizeDemoRows` before reading `APP._sdBrands`.
- **CSV export prepends UTF-8 BOM** (`﻿`) — intentional for Excel on Windows.
- **Image URLs constructed as** `https://view.shelfwatch.io?url=${row.fp}` — `file_path` is the GCS URL.
- **`buildDrillContent` takes 3 args**: `(errors, errorCount, colspan)`. `errors` is `{ brandName: { count, imgs: { sid: {...} } } }`. Do not pass a separate `errorImgs` arg — that was removed.
- **`getPredBrandLabel(r)` must be used** whenever mapping a prediction row to a brand key for error accumulation. Never use `r.pred` directly as the drill-down key.

---

## Section 11: Session Timeline

| Date | What Was Built / Fixed / Validated |
|------|-----------------------------------|
| ~June 2025 | Initial scaffold: 3-file upload, column mapping modal, processing pipeline, masterdata pivot parser, quality normalization |
| ~June 2025 | KPI cards, Chart 1 (quality bar), Chart 2 (brand/variant/SKU grouped bar), Brand Accuracy and Variant Accuracy pivot tables |
| ~June 2025 | SKU Accuracy table with drill-down, Brand Demo sheet (full error taxonomy), Size Demo sheet, sidebar filters, pagination, CSV export, image viewer links |
| ~June 2025 | Quality join warning banner, top-bar stats, fixed "Others" bucket logic, deferred MSL feature |
| June 2025 | **Session 2:** Headline stat cards (Images, Brands, Variants, Groups); 7-tab layout (Overview, Brand, Variant, SKU, Brand Demo, Size Demo, Trends); separate top-15 bar charts per tab (c3, c4); lazy tab rendering to fix hidden canvas sizing |
| June 2025 | Model ID + Date inputs on upload screen; localStorage trend persistence (HISTORY_KEY); Trends tab with history table + 4 line charts (c5–c8); brand/variant/SKU trend selectors; saveSnapshot / deleteSnapshot / clearHistory |
| June 2025 | Brand-level drill-down aggregation (getPredBrandLabel helper); errors restructured to embed image maps per brand; buildDrillContent rewritten to 3-arg signature; all 3 callers updated; inline image chip links (.img-chip) with quality badge in drill-downs |
| June 2025 | Brand Demo switched to top-15 dynamic competitor columns (getTopCompetitorBrandsForDemo); all filters apply across all tabs via filteredRows() + renderActiveTabContent() |
| June 2025 | Deployed to GitHub Pages: https://kartik-15.github.io/cv-accuracy-dashboard/ (repo: Kartik-15/cv-accuracy-dashboard; .nojekyll added) |

---

## Section 12: Context Recovery Prompt

```
I'm working on a single-file HTML browser app at:
  /Users/kartik/Desktop/Accuracy Dashboard/index.html

Deployed at: https://kartik-15.github.io/cv-accuracy-dashboard/
GitHub repo: https://github.com/Kartik-15/cv-accuracy-dashboard

PROJECT: CV Accuracy Dashboard for Coca-Cola India (CCI). Analyzes computer vision model
accuracy on NARTD cooler shelf bounding-box predictions. 100% client-side, no server.

STACK: Vanilla JS + Tailwind CDN + Chart.js + PapaParse + SheetJS. No build tools.

INPUTS (3 file upload slots + model ID + date):
  1. Detection Data (XLSX "Raw" sheet or CSV):
     Required cols: session_id, QC_Display Name (GT), Display Name (Pred), competition
     Optional: file_path (GCS URL for image chips), quality (embedded fallback)
  2. Image Quality (CSV): cooler_session_id → image_quality
  3. Masterdata (CSV, CGC pivot format): class_name + attribute_name + attribute_value
  4. Model ID (text input) + Date (date picker) — stored per run for trend tracking

KEY RULES:
  - is_correct = (QC_Display Name === Display Name) — always recomputed, never from file
  - competition column in Raw = PREDICTION's type; GT competition from masterdata only
  - Masterdata keys: displayName.toLowerCase()
  - Quality normalization: "Good Quality"→"Good", "Blurred image"→"Blurred", etc.
  - Brand Demo: GT 'self' items only; top-15 competitor columns by error frequency (dynamic)
  - Size Demo: same-brand, different-measure errors only (both measures non-zero)
  - KPI thresholds: ≥90%=green, 75-89%=amber, <75%=red
  - Drill-downs aggregate by brand name via getPredBrandLabel(r), not Display Name
  - buildDrillContent(errors, total, colspan) — 3 args; errors = { brandName: { count, imgs:{} } }
  - Trend snapshots saved to localStorage key 'cci_accuracy_history_v1' after each processData()
  - Lazy tab rendering: renderActiveTabContent() called on switchMainTab()
  - Chart instances c1–c8; all destroyed in resetAll()

APP STATE:
APP = {
  files, raw, cfg,
  d: { rows, qMap, mMap, brands, qualities, variants, measures, subcats, modelId, date },
  f: { qualities:[], brands:[], variants:[], measures:[] },
  pg: { bd:{page,size,q,rows}, sd:{page,size,q,rows} },
  tabs: { main: 'overview' },
  charts: { c1, c2, c3, c4, c5, c6, c7, c8 },
  _sdBrands, _skuExpanded, _skuQ, _pivotSku,
  _trendBrands, _trendVariants, _trendSkus,
  _brandExpanded, _variantExpanded, _pivotBrand, _pivotVariant
}

CURRENT STATUS: Core features + trend tracking complete. Deployed to GitHub Pages.
NEXT FEATURE: MSL — 4th file upload slot, join on store/session ID, compliance columns in Size Demo.

SAMPLE DATA FILES in same folder:
  - Accuracy Sheet.xlsx (detection data — use "Raw" sheet)
  - CCI_Image_Quality_Analysis - 12th June - Sheet1.csv (quality)
  - CGC_New_15th June.csv (CANONICAL masterdata)
```
