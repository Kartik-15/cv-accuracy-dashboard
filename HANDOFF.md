# CV Accuracy Dashboard — Handoff Document

---

## Section 1: Quick Start

### What It Does
A single-file browser app (`index.html`) for analyzing computer vision model accuracy on NARTD cooler shelf images. CV/QC teams upload three files — bounding-box detection data, image quality mapping, and product masterdata — and get back accuracy KPIs, charts, drill-downs, and per-image breakdown tables. Zero server involvement; all computation runs client-side.

### Hosted URL
**https://kartik-15.github.io/cv-accuracy-dashboard/**

GitHub repository: `https://github.com/Kartik-15/cv-accuracy-dashboard`
Deployment: GitHub Pages, served from `main` branch root. `.nojekyll` file prevents Jekyll from intercepting the single HTML file.

### File/Module Status

| File | Status | Note |
|------|--------|------|
| `index.html` | **ACTIVE** | Entire app — HTML + CSS + JS in one file (~3300+ lines) |
| `.nojekyll` | **ACTIVE** | Required for GitHub Pages to serve `index.html` directly |
| `CGC_New_15th June.csv` | **ACTIVE** | Canonical masterdata sample — Coolers/Rollout project |
| `Accuracy Sheet.xlsx` | **ACTIVE** | Sample detection data (use `Raw` sheet) |
| `CCI_Image_Quality_Analysis - 12th June - Sheet1.csv` | **ACTIVE** | Sample image quality file (now optional) |
| `New/CCI_Warm_Shelves_Accuracy.xlsx` | **ACTIVE** | Warm Shelves detection data (sheet: `CCI_WS_ai acc_June 15`) |
| `New/WarmShelves_Masterdata.csv` | **ACTIVE** | Generated masterdata for Warm Shelves project (Display Name remapped to class_name) |
| `New/fe534025-dcb1-4cec-a665-1b3200c9a9aa_groupclassinfo.csv` | **REFERENCE** | Raw groupclassinfo export — Warm Shelves classes; Display Name is in attribute_value where attribute_name='Display Name', NOT in class_name directly |
| `CCI_Rollout_CGC_15th June.csv` | **SUPERSEDED** | Older masterdata; replaced by `CGC_New` |
| `Final Brand and Variant.csv` | **SUPERSEDED** | Pre-flattened masterdata; app now builds this map internally |
| `RA Accuracy.xlsx` | **UTILITY** | Earlier accuracy sheet; same schema |

### Current Objective
SKU Analysis tab bugs fixed (session 8): duplicate rows resolved, all error images now shown, self-error phantom entries eliminated. Next: export CSV for SKU accuracy table, MSL feature, and Trends tab polish.

### Immediate Next Steps
1. Verify multi-model upload flow end-to-end with real files (2+ models, same masterdata).
2. SKU Analysis tab: export CSV with image links (mirrors Brand Accuracy export pattern).
3. Implement MSL feature: add a file upload zone for store-level MSL file, join on store/session ID, append MSL compliance columns to the Size Demo table.
4. Test with larger datasets to identify in-browser performance limits.

### Files Most Likely to Be Edited Next

| Path | Why |
|------|-----|
| `index.html` `renderSkuAccuracy` / `buildSkuAccuracy` | Add export CSV button (image URLs per error SKU) |
| `index.html` `renderTrendsTab` / `renderComparisonSection` | Polish multi-model comparison chart |
| `index.html` upload screen | Add zone for MSL file (future) |
| `index.html` `buildSizeDemoRows` / `renderSizeDemo` | Inject MSL compliance columns (future) |

---

## Section 2: Project Overview

### Full Capabilities (current state)

1. **Multi-model upload** — Multiple Detection Data files (one per model run), each with a user-typed model name. Image Quality (optional) and Masterdata are shared across all models. Drag-and-drop or click-to-browse. "+ Add Model" button adds slots dynamically.
2. **Auto date extraction** — Date is auto-read from the `date` column in each detection file; no manual date entry. Model ID + Date inputs removed from upload screen.
3. **Flexible column mapping modal** — auto-detects column names via candidate list matching; users can override via dropdown before processing. Handles XLSX sheet selection.
4. **Quality normalization** — maps varied strings ("Good Quality", "Blurred image", "blank image") to canonical tiers: Good / Average / Poor / Blank / Blurred / Impossible / Unknown.
5. **Masterdata pivot parsing** — converts CGC-format rows (`class_name`, `attribute_name`, `attribute_value`) into a flat `displayName → {brand, variant, measure, subcat, group, class_name, competition, attrs}` lookup map. `group_name` is read as a **direct CSV column** (not an attribute pair); `class_name` is the pivot key.
6. **Enriched bounding-box rows** — joins quality and masterdata attributes onto each detection row; computes `is_correct` fresh from GT vs Prediction string equality. Each row carries `gt_group` (from `group_name` column) and `gt_class` (from `class_name` pivot key).
7. **Headline stat cards** — 5 cards with descriptive subtitles: Images, Unique Brands, Unique Groups (from `group_name` direct column), Unique Classes (from `class_name` pivot key), Unique SKUs (unique GT Display Names matched to masterdata). Always visible, filter-reactive.
8. **KPI cards** — Object Accuracy, Macro Precision, Macro Recall, Macro F1 with color-coded threshold badges (≥90% emerald, 75–89% amber, <75% red).
9. **Multi-tab layout** — 4 main tabs: Brand Analysis (default), Variant Analysis, SKU Analysis, Trends. Brand Demo is embedded in Brand Analysis; Size Demo is embedded in SKU Analysis. Overview tab removed.
10. **Lazy tab rendering** — only the active tab's charts/tables re-render on filter change, avoiding hidden-canvas sizing bugs.
11. **Brand Analysis tab** — layout order: (1) Brand Accuracy table, (2) Brand Demo table, (3) Breakdown charts at bottom. No top chart (c3 removed).
12. **Brand Accuracy table** — Self/Competitor filter toggle; Export CSV button (exports all errors with image session IDs and image URLs per misclassified brand); two-level drill-down: click brand row → see error brand breakdown (counts only), click image icon on error brand → expand full image list for that brand.
13. **Brand Demo table** — Self Misclassification = GT is 'self', pred is a *different* self brand (`r.comp === 'self' && gt_brand !== pred_brand`). Same-brand different-SKU falls to Others. Self/Competitor filter toggle. Paginated (25/page), searchable, CSV exportable.
14. **Variant Analysis tab** — Fully overhauled (session 6). Attribute selector dropdown (pick any masterdata attribute, not just variant). Granularity toggle: "Brand+Attr" (e.g. "MAAZA · PET") vs "Attr only" (e.g. "PET"). Self/Competitor filter toggle. Export CSV button. Two-level drill-down: level 1 shows which attribute values it was misclassified as; level 2 expands images. Breakdown charts at bottom (one chart per checked filter value, shared label order — mirrors Brand Analysis pattern).
15. **SKU Analysis tab** — Layout (top to bottom): (1) SKU Coverage card, (2) SKU Accuracy table, (3) Size Demo card.
16. **SKU Coverage card** — Shows masterdata universe vs. accuracy-sheet coverage split by All / Self / Competitor. Counts from all uploaded rows (not sidebar-filtered) so coverage reflects the dataset, not a filter slice. Animated progress bar per segment; color-coded green ≥90% / amber ≥70% / red <70%. Warning banner when masterdata SKUs have zero GT observations.
17. **SKU Accuracy table** — Self/Competitor/All filter toggle (`_skuAccuracyFilter`). Accuracy bucket stat chips above the table (Total / >90% / 80–90% / 60–80% / <60%) — each chip is also a clickable filter (`_skuBucketFilter`). Bucket counts are computed after the Self/Comp filter but before bucket filter, so they always reflect the tier breakdown for the current Self/Comp scope. Click-to-expand drill-down shows **predicted Display Name** (SKU-level), not brand. Up to 6 image chips per predicted SKU.
18. **Size Demo card** (inside SKU Analysis tab) — Image-level aggregation for same-brand wrong-size errors. Paginated, searchable, CSV exportable. All columns sortable.
17. **Trends tab** — Two sections: (1) **Model Comparison** — live comparison chart of all models uploaded this session: X = dates (from file), Y = accuracy %, one line per model name; toggle Brand/Variant/SKU; Self/Competitor filter; comparison table (rows = models, columns = dates). (2) **History** — localStorage-persisted KPI line chart (c5) + Brand/Variant/SKU trend selectors + charts (c6–c8); history table with per-run delete + Clear All. Key: `cci_accuracy_history_v1`.
18. **Sidebar filters** — Fully dynamic from masterdata attributes. One filter section per attribute key plus Image Quality. All filters apply across all tabs simultaneously via `filteredRows()`. Reset All button. Empty/unknown/N/A values excluded via `_isBlankVal()`.
19. **Configurable filter section picker** — Pill buttons at top of sidebar toggle which attribute filter sections are visible. Preference saved to localStorage (`cci_filter_sections_v2`). Default visible: Image Quality + Brand.
20. **Sortable columns** — All tables have clickable column headers. Sort state lives in `APP._sort`.
21. **Light/dark theme toggle** — Sun/moon button in top bar and upload screen. Saves to localStorage (`cci_theme`). CSS custom properties used throughout.
22. **Breakdown brand accuracy charts** — At the bottom of Brand Analysis tab. Checking any filter value renders one Brand Accuracy Top-15 chart per checked value. All charts share the same brand order (global top-15 by combined volume). Charts disappear when no checkboxes are checked.
23. **Brand/Variant accuracy correctness** — Brand accuracy = `pred_brand === gt_brand`. Variant = brand + variant match. SKU = exact `is_correct`. "Unknown" rows excluded.
24. **Image viewer links** — session IDs rendered as `.img-chip` elements linking to `https://view.shelfwatch.io?url={file_path}`.
25. **Brand-agnostic labels** — All "Self" brand labels shown as `Self` (not hardcoded as "CCI") throughout the UI, making the dashboard reusable for any client.

### Current Status
**Deployed** at https://kartik-15.github.io/cv-accuracy-dashboard/ — shareable with the team. All core features including trend tracking, dynamic filters, Brand Analysis overhaul, and light/dark theme are live.

### Key Constraints and Non-Obvious Rules
- `is_correct` is **always recomputed** as `QC_Display Name === Display Name` string equality. The `DN_Acc` column in the XLSX is deliberately ignored — it was found to be unreliable.
- **Brand accuracy uses `pred_brand === gt_brand`**, not `is_correct`. Same-brand wrong-SKU is correct at brand level.
- **Variant accuracy uses `pred_brand === gt_brand && pred_variant === gt_variant`** (with fallback when gt_variant is empty).
- **SKU accuracy uses exact `is_correct`** — the only level where exact Display Name match is the right metric.
- The `competition` column in the Raw detection sheet reflects the **prediction's** competition type, not the GT item's. GT competition must be looked up from the masterdata map.
- **`group_name` is a direct CSV column** in the CGC masterdata, NOT an attribute_name/attribute_value pair. Read via `row['group_name']` in `buildMasterdataMap`, stored as `attrs._gname`, then mapped to `entry.group`.
- Brand Demo only includes bounding boxes where the **GT item is 'self'** brand.
- **Self Misclassification in Brand Demo** = GT is self, pred is a *different* self brand (`r.comp === 'self' && r.gt_brand !== r.pred_brand`). Same-brand different-SKU goes to Others, not Self Misclass.
- In Brand Demo, items with an unknown `pred_brand` fall into the **Others** bucket.
- Size Demo only captures errors where **GT brand === Pred brand** AND **GT measure ≠ Pred measure** AND both measures are non-zero.
- Masterdata key lookup is always **case-insensitive** (`displayName.toLowerCase()`).
- Brands excluded from the unique-brand column list: empty string, null, `'Others'`, `'Non-NARTD'`, `'#N/A'`.
- Quality fallback order: IQ file lookup → embedded `quality` column in detection file → `'Unknown'`.
- CSV export prepends a UTF-8 BOM (`﻿`) so Excel opens it without encoding issues.
- The Detection Data XLSX defaults to the `Raw` sheet if it exists; otherwise falls back to the first sheet.
- Trend data persists across page loads in localStorage but is cleared if the user manually clears browser storage.
- `_isBlankVal(v)` is the universal check for empty/placeholder values — used in filter list building and table row skipping.

---

## Section 3: Core Logic / Algorithm

### is_correct Computation
```js
const is_correct = gt !== '' && gt === pred;
// gt  = cl(row[cols.qc_dn])   — QC Display Name (Ground Truth)
// pred = cl(row[cols.dn])     — Display Name (AI Prediction)
// cl() = v => String(v).trim()
```

### Accuracy Level Definitions
```
SKU-level correct:     is_correct === true  (exact Display Name match)
Brand-level correct:   pred_brand === gt_brand
Variant-level correct: pred_brand === gt_brand && (gt_variant === '' || pred_variant === gt_variant)
```

### Masterdata Map Build (`buildMasterdataMap`)
```js
// Input: CGC pivot rows — columns include class_name, group_name (direct), attribute_name, attribute_value, competition
// Output: { "display name lowercase": {brand, variant, measure, subcat, group, class_name, competition, attrs} }

for each row:
  cname = row[cols.class_name]          // direct column
  gname = row['group_name']             // direct column — NOT an attribute pair
  store gname in byClass[cname]._gname
  collect attribute_name:attribute_value pairs into byClass[cname]

for each class entry:
  dn = attrs['Display Name'] || attrs['display_name']  → map key
  group = attrs._gname || attrs['group_name'] || ...   → populated from direct column
  store all remaining attrs in entry.attrs for dynamic filters
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
| 2 | Non-NARTD | `pred === 'Non-NARTD'` |
| 3 | Others | `pred === 'Others'` |
| 4 | Self Misclassification | `r.comp === 'self' && gt_brand !== pred_brand` — different self brand |
| 5 | [Brand X] | `pred_brand` is a known competitor brand in top-15 columns |
| 6 | Others (fallback) | same brand different SKU, or unknown pred_brand |

**Key change from session 5**: Self Misclass now requires `gt_brand !== pred_brand` (brand-level cross-self error). Same-brand-different-SKU errors no longer count as Self Misclass — they fall through to Others.

### Brand Accuracy Drill-Down (two-level)
```js
// Level 1: Click brand row → buildBrandDrillContent()
//   Shows: Predicted Brand | Count | % of Errors | Images (count only)
//   Each error-brand row has an image icon button

// Level 2: Click image icon → toggleBrandErrorImg(brandName, errorBrand)
//   Toggles APP._brandErrorImgExpanded["brandName||errorBrand"]
//   When expanded: shows ALL images for that specific error brand (no cap)
//   Uses buildImgChips() for image rendering

// State: APP._brandErrorImgExpanded = { "BrandA||BrandB": true, ... }
// Reset: on filter change (setBrandAccuracyFilter) and on resetAll()
```

### Breakdown Charts Logic (`renderBrandBreakdownCharts`)
```js
// Triggered when any filter checkbox is checked
// Renders at the BOTTOM of Brand Analysis tab (after Brand Demo)
// For each active filter section with checked values:
//   - Get selected values from APP.f.qualities or APP.f.attrs[attrName]
//   - For each selected value, filter APP.d.rows to that value only
//   - Compute _brandStats() per sub-chart
//   - Compute globalOrder: top-15 brands by combined volume across all checked values
//   - Render one chart per checked value, all using globalOrder for consistent y-axis
// Charts stored in APP._breakdownCharts; destroyed before each re-render
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
1. At least one detection file staged + masterdata staged → **Process button enabled** (IQ optional, no model ID / date required)
2. Column mapping confirmed, all required fields mapped → **Processing starts**
3. `buildMasterdataMap` completes → `APP.d.mMap` populated
4. `buildQualityMap` completes → `APP.d.qMap` populated
5. `enrichRows` completes → `APP.d.rows` populated
6. Unique filter values extracted (with `_isBlankVal` exclusions applied)
7. `renderAll()` called → `saveSnapshot()` called after render

**Stop conditions:** Missing required column mapping → alert, abort. File parse error → alert, abort.

---

## Section 4: Architecture

### End-to-End Flow

```
┌──────────────────────────────────────────────────────┐
│                   UPLOAD SCREEN                      │
│  Detection Files (multi-slot):                       │
│    [File zone] [Model name input]  [+ Add Model]     │
│    [File zone] [Model name input]  ...               │
│  [Image Quality CSV]  ← Optional                     │
│  [Masterdata CSV]                                    │
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
│  extract unique filter values (_isBlankVal excluded) │
│  renderAll() → saveSnapshot()                        │
└──────────────┬───────────────────────────────────────┘
               │
               ▼
┌──────────────────────────────────────────────────────┐
│                   DASHBOARD                          │
│                                                      │
│  filteredRows()  ← sidebar filter state (APP.f)     │
│       │                                              │
│       ├─ renderHeadlineStats() → 5 stat cards        │
│       ├─ renderKPIs()          → 4 KPI cards         │
│       ├─ renderFilters()       → sidebar update      │
│       └─ renderActiveTabContent(rows)                │
│             │                                        │
│             ├─ brand:   Brand Accuracy table         │
│             │           + Brand Demo table           │
│             │           + breakdown charts (bottom)  │
│             ├─ variant: attr selector + pivot + breakdown charts │
│             ├─ sku:     SKU accuracy table + Size Demo│
│             └─ trends:  comparison chart + c5–c8 + history table │
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
  gt_group:     string,   // group_name of GT item (from direct CSV column in masterdata)
  gt_class:     string,   // class_name of GT item (masterdata pivot key)
  gt_comp:      string,   // 'self'|'competitor' — GT item's competition (from masterdata)
  gt_attrs:     object,   // all raw masterdata attrs for GT item (for dynamic filters)
  pred_brand:   string,   // Brand of predicted item (from masterdata)
  pred_variant: string,   // Pack type of predicted item
  pred_measure: number,   // Size in ml of predicted item
  _model:       string,   // Model name (from upload slot text input) — NEW session 6
  _fileDate:    string,   // YYYY-MM-DD extracted from 'date' column in detection file — NEW session 6
}

// Masterdata map (APP.d.mMap)
{ "display name lowercase": { brand, variant, measure, subcat, group, class_name, competition, attrs:{} } }
// Note: group comes from group_name direct column, stored via attrs._gname during build

// Quality map (APP.d.qMap)
Map<sessionId: string, quality: string>

// Error structure in buildBrandAccuracy / buildVariantAccuracy / buildSkuAccuracy
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
  files: { dets:[{file,modelName}], iq, md },  // dets = array, one entry per model slot
  raw:   { dets:[], iq, md },                  // parsed: { isXLSX, sheets, activeSheet, data }
  cfg:   { det, iq, md },                      // { sheet: string, cols: { key: columnName } }
  d: {
    rows, qMap, mMap,
    brands, qualities, variants, measures, subcats,
    attrKeys: string[],               // sorted list of all masterdata attribute keys
    attrValues: { [key]: string[] },  // unique non-blank values per attribute
    modelNames: string[],             // unique model names in insertion order
    dateRange: string,                // summary string e.g. "3 dates"
    modelId: string,                  // legacy — empty string
    date: string                      // legacy — empty string
  },
  f:     { qualities:[], brands:[], variants:[], measures:[], attrs:{} },
  pg:    { bd: {page,size,q,rows}, sd: {page,size,q,rows} },
  tabs:  { main: 'brand' },
  charts: { c1:null, c2:null, c3:null, c4:null, c5:null, c6:null, c7:null, c8:null },
  _sort: { brand:{col,dir}, variant:{col,dir}, sku:{col,dir}, bd:{col,dir}, sd:{col,dir} },
  _visibleFilterSections: Set,
  _breakdownCharts: {},         // brand breakdown chart instances
  _variantBreakdownCharts: {},  // variant breakdown chart instances
  _brandAccuracyFilter: 'all',
  _brandDemoFilter: 'all',
  _brandErrorImgExpanded: {},   // { "brandName||errorBrand": true }
  _variantAccuracyFilter: 'all',
  _variantAttrKey: string,      // currently selected attribute for Variant tab
  _variantGroupBy: 'brand+attr',// 'brand+attr' | 'attr'
  _variantErrorImgExpanded: {}, // { "rowLabel||errorVariant": true }
  _activeModel: 'all',          // model filter for Brand/Variant/SKU tabs
  _trendsAccLevel: 'brand',     // 'brand'|'variant'|'sku' — Trends tab accuracy level
  _trendsFilter: 'all',         // 'all'|'self'|'competitor'
  _trendsCompChart: null,       // Chart.js instance for model comparison chart
  _sdBrands: [],
  _skuExpanded: Set,
  _skuQ: string,
  _pivotSku: [],
  _skuAccuracyFilter: 'all',  // 'all'|'self'|'competitor'
  _skuBucketFilter: 'all',    // 'all'|'gt90'|'80to90'|'60to80'|'lt60'
  _trendBrands: [],
  _trendVariants: [],
  _trendSkus: [],
  _brandExpanded: Set,
  _variantExpanded: Set,
  _pivotBrand: [],
  _pivotVariant: []
}
```

### File/Module Map

```
Accuracy Dashboard/
├── index.html                    ACTIVE — entire app (HTML + CSS + JS, ~2700+ lines)
│   ├── CSS (.main-tab, .main-tab.active, .drill-wrap, .img-chip, .hstat* classes;
│   │        CSS vars: --section-label, --code-bg, --code-text)
│   ├── Upload Screen (#uploadScreen)
│   │   └── Multi-slot detection zone (#detSlotsContainer) + addModelSlot/removeModelSlot
│   ├── Column Mapping Modal (#mappingModal)
│   ├── Processing Overlay (#procOverlay)
│   ├── Dashboard (#dashboard)
│   │   ├── Top bar (model ID + date + images + quality-matched stats, re-upload, theme toggle)
│   │   ├── Headline stat cards (Images, Brands, Groups, Classes, SKUs — each with subtitle)
│   │   ├── Main tab nav (Brand | Variant | SKU | Trends)  ← no Overview
│   │   ├── Sidebar (filter section picker pills + dynamic attribute filter sections)
│   │   └── Tab content divs
│   │       ├── Brand tab layout: #baFilterBar → baHead/baBody → Brand Demo → #brandBreakdownCharts
│   │       └── SKU tab layout: #skuCoverageCard → #skuFilterBar + #skuStatChips + #skuSearch → skuHead/skuBody → Size Demo
│   └── JavaScript
│       ├── APP state object
│       ├── _BLANK_VALS / _isBlankVal(v)  — universal empty-value guard
│       ├── HISTORY_KEY + localStorage trend persistence
│       ├── File staging & drag-drop
│       ├── parseXLSX / parseCSV / parseFileAuto
│       ├── normalizeQuality
│       ├── COL_DEFS + bestMatch (column auto-detection)
│       ├── buildTabHTML / switchTab / openMappingModal
│       ├── processData (pipeline orchestrator; reads modelId/date; calls saveSnapshot)
│       ├── buildMasterdataMap / buildQualityMap / enrichRows
│       ├── filteredRows / filteredRowsExcept(excludeKey)
│       ├── renderFilters / toggleFilter / toggleAttrFilter
│       ├── loadFilterSections / buildFilterSectionDefs / getAttrStyle
│       ├── sortTable / applySort / sTh  (sortable column helpers)
│       ├── setBrandAccuracyFilter / setBrandDemoFilter / renderFilterToggle  ← NEW
│       ├── getChartDefaults()  (theme-aware chart colors)
│       ├── applyTheme / toggleTheme / initTheme
│       ├── computeMetrics
│       ├── renderHeadlineStats / renderKPIs
│       ├── MAIN_TABS / switchMainTab / renderActiveTabContent
│       ├── renderChart1 (dead — overview removed) / renderChart2 (dead)
│       ├── getPredBrandLabel (maps pred row → brand label)
│       ├── _brandStats(rows)  — brand stats map helper for breakdown charts
│       ├── buildBrandAccuracy / renderBrandAccuracy
│       ├── exportBrandAccuracy()  ← NEW — CSV with image URLs per error brand
│       ├── toggleBrandDrill / toggleBrandErrorImg  ← toggleBrandErrorImg is NEW
│       ├── buildDrillContent (errors, total, colspan) — used by Variant + SKU tabs
│       ├── buildBrandDrillContent (brandName, errors, total, colspan)  ← NEW — Brand tab only
│       ├── buildImgChips / renderBrandSubChart(canvasId, rows, fixedLabels)
│       ├── renderBrandBreakdownCharts()
│       ├── buildAttrAccuracy(rows, attrKey, groupBy)  ← NEW — core Variant tab metric builder
│       ├── _attrStats(rows, attrKey, groupBy)         ← NEW — mirrors _brandStats for variant charts
│       ├── renderVariantAccuracy / renderVariantBreakdownCharts  ← overhauled session 6
│       ├── _modelFilteredRows()                       ← NEW — filters rows by APP._activeModel
│       ├── setActiveModel(name)                       ← NEW — sets model filter + re-renders
│       ├── renderModelPillBar(containerId)            ← NEW — renders model selector pills
│       ├── renderTrendsTab / renderComparisonSection  ← overhauled session 6
│       ├── renderKpiTrendChart (c5) / renderHistoryTable
│       ├── renderSkuCoverageCard()                    ← NEW session 7 — masterdata vs detection coverage
│       ├── buildSkuAccuracy / renderSkuAccuracy / toggleSkuDrillIdx
│       ├── setSkuAccuracyFilter(val) / setSkuBucketFilter(val)  ← NEW session 7
│       ├── getTopCompetitorBrandsForDemo / buildBrandDemoRows / renderBrandDemo
│       ├── buildSizeDemoRows / renderSizeDemo
│       ├── renderPag / changePg / exportCSV (brand_demo_sheet + size_demo_sheet)
│       ├── saveSnapshot / deleteSnapshot / clearHistory
│       ├── renderTrendsTab / renderHistoryTable
│       ├── renderKpiTrendChart (c5)
│       ├── renderBrandTrendSelector / renderBrandTrendChart (c6)
│       ├── renderVariantTrendSelector / renderVariantTrendChart (c7)
│       ├── renderSkuTrendSelector / renderSkuTrendChart (c8)
│       ├── toggleTrendItem(type, item)
│       └── resetAll (destroys charts, resets all filter/expand state including new session-5 state)
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
| Masterdata | CSV (CGC pivot) | `class_name`, `attribute_name`, `attribute_value`, `group_name` (direct col) |

Optional detection columns: `file_path` (for image viewer chips), `quality` (embedded quality fallback).

### KPI Output Schema

| Metric | Formula | Color Threshold |
|--------|---------|----------------|
| Object Accuracy | correct / total | ≥90% green, 75–89% amber, <75% red |
| Macro Precision | mean(TP/predN per class) | same |
| Macro Recall | mean(TP/gtN per class) | same |
| Macro F1 | 2·P·R / (P+R) | same |

### Headline Stats (always visible, filter-reactive)

| Card | Value | Subtitle |
|------|-------|----------|
| Images | Distinct `session_id` values | "Unique shelf images" |
| Unique Brands | Distinct `gt_brand` values | "GT brand names" |
| Unique Groups | Distinct `gt_group` values (from `group_name` direct column) | "Brand groups in data" |
| Unique Classes | Distinct `gt_class` values (from `class_name` pivot key) | "Product class types" |
| Unique SKUs | Distinct `r.gt` where `r.gt_class` is non-empty | "Matched GT display names" |

### Brand Accuracy Export (`brand_accuracy.csv`)

Flat rows — one row per (GT Brand × Predicted Brand) error combination:

| Column | Definition |
|--------|-----------|
| GT Brand | Ground-truth brand name |
| Type | `Self` or `Competitor` |
| Total | Total GT bboxes for this brand |
| Correct | Correctly predicted bboxes |
| Errors | Total error count |
| Accuracy % | `correct / total × 100` |
| Predicted As | The brand it was misclassified as |
| Error Count | Count for this specific misclassification |
| % of GT Errors | Share of this brand's total errors |
| Image Session IDs | Semicolon-separated session IDs |
| Image URLs | Semicolon-separated ShelfWatch viewer URLs |

### Brand Demo Export (`brand_demo_sheet.csv`)

| Column | Definition |
|--------|-----------|
| Image ID | `session_id` |
| Image Quality | Normalized quality tier |
| Self Facings | Correct self-brand predictions in this image |
| Self Misclassification | Self GT predicted as a different self brand (brand-level cross-self error) |
| Non-NARTD | Predicted as `Non-NARTD` |
| Others | Predicted as `Others` OR same-brand different-SKU OR unknown brand |
| [Brand X] × top 15 | Count predicted as this competitor brand (dynamic columns) |
| Total Self GT | Total self-brand GT bounding boxes in this image |

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
- The Brand Demo sheet is for self-brand analysis. Competitor GT items are out of scope.

**4. Unrecognized pred_brand falls into Others in Brand Demo**
- Edge case where masterdata doesn't cover a prediction label — lumping into Others is safer than creating phantom columns.

**5. Drill-downs aggregate by brand name, not Display Name**
- `getPredBrandLabel(r)` maps any prediction to its brand; error accumulator uses brand as key.
- Users want "was Brand A mistaken as Brand B?" not a list of 40 individual SKUs.
- `buildDrillContent(errors, total, colspan)` — 3 args; `errors = { brandName: { count, imgs:{} } }`.

**6. Brand Demo uses top-15 competitor brands by actual error frequency, not all masterdata brands**
- `getTopCompetitorBrandsForDemo(rows, 15)` counts competitor brand predictions in filtered data; top 15 become columns.
- Dynamic top-15 keeps columns meaningful and filter-reactive.

**7. Lazy tab rendering via `renderActiveTabContent()`**
- Chart.js cannot measure canvas dimensions when the canvas is hidden. Rendering only on activation ensures correct sizing.

**8. Trend snapshots keyed by `${modelId}__${date}`**
- Double-underscore separator chosen to be unlikely to appear in model IDs or dates. Snapshots overwritten on re-run with same key.

**9. Trend snapshot stores only top-150 SKUs**
- Full SKU list can be thousands of entries. localStorage has a 5–10 MB limit.

**10. MSL feature deferred**
- Requires a 4th file upload with store-level MSL data; out of scope for current version.

**11. CGC pivot format as canonical masterdata**
- `CGC_New_15th June.csv` (pivot format) is canonical. `Final Brand and Variant.csv` (pre-flattened) is superseded.

**12. Brand Demo and Size Demo merged into Brand/SKU tabs**
- Fewer tabs reduces navigation overhead; demo sheets are contextually related to their parent accuracy tables.

**13. Filters fully dynamic from masterdata attributes**
- `buildMasterdataMap` stores all raw attributes in `entry.attrs`. One filter section per attribute key. `filteredRows` filters on `r.gt_attrs[attrName]`.
- localStorage key: `cci_filter_sections_v2`.

**14. Sort state is per-table, not global**
- Each table has its own `{col, dir}` entry in `APP._sort`.

**15. Light/dark theme via CSS custom properties + body.light class**
- `body.light` block overrides all `--var` values. `getChartDefaults()` returns theme-appropriate colors each time charts are created.

**16. Brand/variant accuracy uses brand-level match, not exact SKU match**
- Brand accuracy = `pred_brand === gt_brand`; variant = brand + variant both match; SKU = `is_correct`.
- Same-brand wrong-SKU disappears from brand drill-downs; brand accuracy reflects true brand identification rate.

**17. Breakdown charts at bottom of Brand Analysis tab**
- Charts moved to after the Brand Demo table (session 5). Previously between top chart and Brand Accuracy table.
- Appear on demand when filter checkboxes are checked.

**18. `_isBlankVal` as universal empty-value guard**
- `_BLANK_VALS` Set + `_isBlankVal(v)` applied at three points: `processData` build, `buildFilterSectionDefs` render, and all three accuracy table builders.

**19. Overview tab removed**
- Removed in session 5. Brand Analysis is now the default landing tab (`MAIN_TABS = ['brand','variant','sku','trends']`).
- Charts c1 and c2 still exist as dead functions; c3 removed from active rendering. All three are still destroyed in `resetAll()` for safety.

**20. Brand Accuracy two-level drill-down**
- Level 1 (click brand row): shows error brand breakdown — counts and image count only, no inline chips.
- Level 2 (click image icon on error brand): expands ALL images for that error (no 6-image cap).
- Separate from `buildDrillContent` which remains for Variant/SKU tabs (still shows inline chips).
- State: `APP._brandErrorImgExpanded` keyed as `"gtBrand||errorBrand"`.

**21. Self Misclassification redefined as brand-level cross-self error**
- Old: same brand, different SKU (`gt_brand === pred_brand`, different Display Name).
- New: different self brand (`r.comp === 'self' && gt_brand !== pred_brand`).
- Same-brand-different-SKU now falls to Others. This matches user expectation that Self Misclass = "another self brand".

**22. `group_name` is a direct CSV column, not an attribute pair**
- CGC masterdata CSV has `group_name` as a top-level column on every row. The old code looked for it only in the attribute_name/attribute_value pairs → always empty → Unique Groups = 0.
- Fix: read `row['group_name']` directly in `buildMasterdataMap`, store as `attrs._gname`.

**23. All "Self" brand labels shown as "Self" not "CCI"**
- Dashboard is brand-agnostic. Type column, filter toggles, and export CSV all use "Self" / "Competitor".

**24. Image Quality file made optional**
- IQ file removed from `checkAllStaged` gate — only detection + masterdata required.
- `openMappingModal` skips parsing IQ if not staged; `confirmProcess` skips IQ column validation.
- Quality fallback chain unchanged: `qMap.get(sid) → embedded quality col → 'Unknown'`.
- `qualWarn` banner only shows when IQ was uploaded but matched 0 rows. Silent when not uploaded.

**25. Variant Analysis tab uses attribute selector, not fixed "variant"**
- `buildAttrAccuracy(rows, attrKey, groupBy)` is the generic metric builder — `attrKey` is any masterdata attribute.
- `groupBy = 'brand+attr'` gives "MAAZA · PET" rows; `groupBy = 'attr'` gives "PET" rows.
- Default is `brand+attr` to show SKU-level breakdown. User can toggle.
- Drill-down level 1 shows which attribute values it was misclassified as; level 2 expands images.

**26. Multi-model upload: one file per model, date auto-read from file**
- `APP.files.dets` is an array of `{file, modelName}` objects. All share one masterdata + one optional IQ.
- Date extracted from `date` column in each detection file (most common value → YYYY-MM-DD).
- Every enriched row tagged with `_model` (model name) and `_fileDate` (YYYY-MM-DD).
- Model ID + Date manual inputs removed from upload screen.

**27. Model filter added to Brand/Variant/SKU tabs**
- `_modelFilteredRows()` wraps `APP.d.rows` with an `_activeModel` filter.
- When only 1 model is loaded, the pill bar is hidden — zero UX overhead for single-model use.
- `setActiveModel(name)` resets pagination + expanded state before re-rendering.

**28. Warm Shelves masterdata compatibility**
- `CGC_New_15th June.csv` and the `fe534025...groupclassinfo.csv` cover the Coolers project. Their `class_name` values do NOT match Warm Shelves detection GT display names.
- For Warm Shelves: the display names live in `attribute_value` where `attribute_name = 'Display Name'` in the groupclassinfo CSV. These must be promoted to `class_name` to work with the dashboard.
- `New/WarmShelves_Masterdata.csv` is the pre-generated correct masterdata for Warm Shelves (252/256 GT match, 98.4%).
- Rule: always verify `class_name` values match GT display names before loading. Use the compatibility check script if unsure.

---

## Section 7: Bugs & Corrections

**1–10.** (see previous sessions)

**11. Unique Groups always showed 0**
- **Problem:** `group_name` is a direct column in the CGC CSV (alongside `class_name`), not stored in the `attribute_name`/`attribute_value` pivot. `buildMasterdataMap` only read pivot pairs, so `attrs['group_name']` was always undefined → `entry.group = ''` → `gt_group = ''` → Unique Groups = 0.
- **Fix:** In first loop of `buildMasterdataMap`, read `row['group_name']` directly and store in `byClass[cname]._gname`. In second loop, use `attrs._gname || attrs['group_name'] || ...`.

**12. Self Misclassification counting same-brand-different-SKU errors**
- **Problem:** Condition was `gt_brand === pred_brand` (same brand, any different SKU). These errors showed up in Self Misclass but not in any competitor column, making the numbers confusing ("mistakes are listed but not visible in other columns").
- **Fix:** Changed to `r.comp === 'self' && r.gt_brand !== r.pred_brand` — only brand-level cross-self errors (e.g. COCA-COLA GT predicted as THUMS UP). Same-brand-different-SKU now falls to Others.

**13. SKU accuracy table showing duplicate rows for same display name**
- **Problem:** `buildSkuAccuracy` keyed its accumulator map by `r.gt.toLowerCase()` (raw GT string). `buildMasterdataMap` double-indexes every entry under both `class_name.toLowerCase()` AND `display_name.toLowerCase()`. Detection rows whose `r.gt` matched the `class_name` form produced a different key than rows whose `r.gt` matched the `display_name` form, creating two separate rows in the table for the same SKU.
- **Fix:** Changed map key to `dispName.toLowerCase()` where `dispName = mMap[sku.toLowerCase()].display_name || sku`. Any `r.gt` that resolves to the same masterdata entry now merges into one row.

**14. SKU drill-down image list capped at 6**
- **Problem:** `buildDrillContent` called `.slice(0, 6)` on the images array, hiding most affected images per predicted SKU.
- **Fix:** Removed `.slice(0, 6)`; all affected images now shown. Updated column header from "Affected Images (up to 6 per sku)" to "Affected Images".

**15. SKU drill-down showing the same SKU as a misclassification target**
- **Problem:** `is_correct` uses raw string equality (`r.gt === r.pred`), but the error accumulator key uses `predEntry.display_name` (canonical form from masterdata). When `r.pred` appeared in the detection file in a different casing or as the `class_name` form, `is_correct = false` even though both GT and prediction resolve to the same canonical SKU — causing the drill-down to list "predicted as THUMS UP PET 250ML" for GT "THUMS UP PET 250ML".
- **Fix:** In `buildSkuAccuracy`, after resolving `predSku`, added `|| predSku.toLowerCase() === key` to the correctness check. Rows where the canonical predicted display name equals the canonical GT key are now counted as correct and excluded from errors. Brand and Variant tabs unaffected — they already use canonical names (`r.pred_brand`, `predEntry.attrs[attrKey]`) for both sides of their correctness checks.

---

## Section 8: Open Tasks

### Critical
- None blocking current use.

### Important
- **Verify multi-model upload end-to-end** with 2+ real files — confirm model filter pills, Trends comparison chart, and row tagging all work correctly.
- **SKU Analysis tab export CSV** — add export button (image URLs per error SKU, mirrors `exportBrandAccuracy` pattern). Self/Comp filter, bucket filter, coverage card, and SKU-level drill-down are all done.
- **MSL feature:** Add file upload zone + MSL file parsing + join on store/session + MSL compliance columns in Size Demo table.

### Nice-to-Have
- Performance testing with large datasets (10k+ bounding boxes).
- Add a Confusion Matrix view (top GT → top predicted).
- Add session-level filter (filter by specific image session IDs).
- Persist last-used column mapping preferences (localStorage).
- Add a "Download All" button that exports all tables in one zip.
- Re-add Overview tab with richer views and charts (deferred by user request).

### Explicitly Out of Scope (confirmed)
- Server-side processing or API backend — by design, all computation stays client-side.
- Multi-user auth or session management.
- MSL columns (deferred to later version).

---

## Section 9: Known Limitations

**1. Trend data stored in browser localStorage**
- If a user clears browser storage or switches browsers/machines, trend history is lost.

**2. All bounding boxes weighted equally in Object Accuracy**
- Object Accuracy is `correct / total` across all bboxes, regardless of image quality or GT brand.

**3. Macro metrics include 'Others' and 'Non-NARTD' as classes**
- Intentional. These labels appear as GT values and are included in Macro P/R/F1 computation.

**4. Size Demo brand columns are dynamic, not fixed**
- Brand columns built from whichever brands have size errors in the filtered data. Filter-reactive by design.

**5. Session ID join is exact-match only**
- If `session_id` in detection data and `cooler_session_id` in image quality file differ by any character, the join fails silently.

**6. Image viewer requires GCS URL format**
- The `.img-chip` viewer link only works if `file_path` is a valid GCS URL recognizable by view.shelfwatch.io.

**7. Top-150 SKU cap in trend snapshots**
- Trend charts for SKU accuracy only cover the top 150 SKUs by error count at snapshot time.

---

## Section 10: Preferences & Conventions

- **Single-file architecture is intentional.** No build tools, no npm, no bundling. Libraries loaded from CDN (Tailwind, FontAwesome, PapaParse, Chart.js, SheetJS).
- **Dark theme is default**; light theme togglable via sun/moon button. CSS custom properties used throughout. New variables `--section-label`, `--code-bg`, `--code-text` added in session 4.
- **Table columns: right-align numbers, left-align labels** — enforced via `.left` class. Don't deviate.
- **KPI color thresholds are hardcoded**: ≥90% = green, 75–89% = amber, <75% = red. Business-defined, not configurable.
- **`cl()` is the universal cell normalizer** — always use `cl(row[colName])` when reading detection data.
- **`_isBlankVal(v)` is the universal empty-value check** — use for filter items and table row guards.
- **Accuracy level semantics**: brand = `pred_brand === gt_brand`; variant = brand + variant match; SKU = `is_correct`. Do not conflate these.
- **Chart.js instances c1–c8 are destroyed and recreated** each time their tab activates. Don't skip destroy or you'll leak canvas contexts. c1/c2/c3 are dead code but still safely handled in resetAll.
- **`APP._breakdownCharts` stores breakdown chart instances** — always destroy all entries before re-rendering.
- **`getChartDefaults()` replaces the old `CHART_DEFAULTS` const** — always call it fresh when constructing a chart config, never cache the result.
- **`resetAll()` resets ALL state** including `_brandAccuracyFilter`, `_brandDemoFilter`, `_brandErrorImgExpanded`.
- **Pagination state resets to 1** whenever filters change or new data is processed.
- **`APP._sdBrands` is a side-effect of `buildSizeDemoRows`** — always call `buildSizeDemoRows` before reading `APP._sdBrands`.
- **CSV export prepends UTF-8 BOM** (`﻿`) — intentional for Excel on Windows.
- **Image URLs constructed as** `https://view.shelfwatch.io?url=${row.fp}` — `file_path` is the GCS URL.
- **`buildDrillContent` takes 3–4 args**: `(errors, errorCount, colspan, label?)`. `label` defaults to `'Brand'`. Pass `'SKU'` from the SKU tab so headers read "Predicted SKU". Used by Variant and SKU tabs (inline image chips). Do NOT use for Brand tab — use `buildBrandDrillContent` instead.
- **`buildBrandDrillContent` takes 4 args**: `(brandName, errors, errorCount, colspan)`. Brand-specific: no inline chips, click-to-expand image list per error brand.
- **`getPredBrandLabel(r)` must be used** whenever mapping a prediction row to a brand key for error accumulation.
- **"Self" not "CCI"** — all user-facing type labels use "Self" / "Competitor". Never hardcode "CCI" as a label.
- **`renderFilterToggle(containerId, currentVal, onFn, options)`** — reusable toggle bar helper. `onFn` is a string function name called with the selected value. Used by both Brand Accuracy and Brand Demo filter bars.

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
| June 2025 | **Session 3:** Brand Demo moved into Brand Analysis tab; Size Demo moved into SKU Analysis tab (MAIN_TABS reduced to 5); sortable columns on all 5 tables via sTh()/applySort()/sortTable(); configurable filter section picker (localStorage key cci_filter_sections_v2) |
| June 2025 | Dynamic attribute filters: all masterdata attrs stored in entry.attrs; one filter section per attr key; filteredRows() filters on r.gt_attrs; fixed stuck-at-rendering bug (attrs:{} missing from APP.f reset) |
| June 2025 | Light/dark theme toggle: body.light class + CSS custom properties; getChartDefaults() replaces CHART_DEFAULTS const; sun/moon button on upload screen and top bar; preference saved to localStorage (cci_theme) |
| June 2025 | USER_GUIDE.md created; HANDOFF.md updated to reflect session 3 changes |
| June 2026 | **Session 4:** Headline stats overhauled — removed Unique Variants, Groups now from `group_name` attr, added Unique Classes (class_name pivot key), added Unique SKUs (masterdata-matched GT display names); grid expanded to 5 columns |
| June 2026 | Masterdata map extended: `buildMasterdataMap` now captures `cname` (class_name) and `group_name` attr; enriched rows gain `gt_group` and `gt_class` fields |
| June 2026 | Light mode readability comprehensive fix: new CSS vars `--section-label`, `--code-bg`, `--code-text`; darkened `--border` and `--muted` in light; replaced all hardcoded light/dark hex colors with CSS vars across all inline styles |
| June 2026 | Brand/variant accuracy correctness fix: brand accuracy = `pred_brand === gt_brand`; variant = brand + variant match; SKU unchanged; same-brand wrong-SKU no longer counted as brand-level error |
| June 2026 | Breakdown brand accuracy charts: `renderBrandBreakdownCharts()`, `renderBrandSubChart()`, `_brandStats()` added; `APP._breakdownCharts` added to state; charts appear only when filter checkboxes are checked; shared brand order for cross-chart comparison |
| June 2026 | Empty/unknown value purge: `_BLANK_VALS` + `_isBlankVal()` added; applied at processData build time, buildFilterSectionDefs render time, and all three accuracy table builders |
| June 2026 | HANDOFF.md updated for session 4 |
| June 2026 | **Session 5:** Overview tab removed; Brand Analysis is now the default landing tab; MAIN_TABS = ['brand','variant','sku','trends'] |
| June 2026 | Brand Analysis tab layout reordered: Brand Accuracy table → Brand Demo table → Breakdown charts (bottom) |
| June 2026 | Brand Accuracy table: Self/Competitor filter toggle (`_brandAccuracyFilter`); Export CSV with image URLs per error brand (`exportBrandAccuracy`); two-level drill-down (`buildBrandDrillContent` + `toggleBrandErrorImg` + `_brandErrorImgExpanded`) — level 1 shows counts, level 2 expands all images |
| June 2026 | Brand Demo: Self Misclassification redefined as brand-level cross-self error (`r.comp === 'self' && gt_brand !== pred_brand`); same-brand-different-SKU now goes to Others; Self/Competitor filter toggle (`_brandDemoFilter`) |
| June 2026 | Fixed Unique Groups = 0: `group_name` is a direct CSV column in CGC masterdata, not an attribute pair; `buildMasterdataMap` now reads it as `row['group_name']` and stores as `attrs._gname` |
| June 2026 | Added descriptive subtitles to all 5 headline stat cards |
| June 2026 | All "CCI" labels replaced with "Self" throughout the UI for brand-agnostic dashboard |
| June 2026 | HANDOFF.md updated for session 5 |
| June 2026 | **Session 6:** Image Quality file made optional — removed from `checkAllStaged` gate; parsing skipped when absent; `qualWarn` banner only shows on IQ upload failure, not when skipped |
| June 2026 | Variant Analysis tab overhauled: attribute selector dropdown (any masterdata attr); granularity toggle (Brand+Attr / Attr only); Self/Competitor filter; Export CSV; two-level drill-down (misclassified attr values → image expand); breakdown charts at bottom (one per checked filter value, shared label order) |
| June 2026 | Multi-model upload: replaced single detection slot with dynamic multi-slot container (`detSlotsContainer`); each slot has file zone + model name text input; `addModelSlot` / `removeModelSlot`; `APP.files.dets` array; Model ID + Date inputs removed |
| June 2026 | Row tagging: every enriched row gets `_model` (typed name) and `_fileDate` (YYYY-MM-DD from date column) |
| June 2026 | Model filter pill bar on Brand/Variant/SKU tabs: `_modelFilteredRows()` + `setActiveModel()`; hidden when only 1 model loaded |
| June 2026 | Trends tab: new Model Comparison section with live comparison chart (one line per model, X=dates, Y=accuracy); toggle Brand/Variant/SKU; comparison table; localStorage history section retained |
| June 2026 | topStats bar updated: shows model count + date range instead of model ID + single date |
| June 2026 | Warm Shelves compatibility investigation: `CGC_New_15th June.csv` covers Coolers only (59/256 GT match for Warm Shelves); generated `New/WarmShelves_Masterdata.csv` by promoting Display Name attribute_value to class_name (252/256 match) |
| June 2026 | HANDOFF.md updated for session 6 |
| June 2026 | **Session 7:** SKU Analysis drill-down changed from brand-level to SKU-level — `buildSkuAccuracy` now groups errors by `r.pred` (predicted Display Name) instead of `getPredBrandLabel(r)` (brand); `buildDrillContent` gains optional 4th `label` param (default `'Brand'`); SKU tab passes `'SKU'` so headers say "Predicted SKU" and "Misclassified as (sku)" |
| June 2026 | SKU Accuracy table: Self/Competitor/All filter toggle (`_skuAccuracyFilter` + `setSkuAccuracyFilter`) — same pattern as Brand/Variant tabs |
| June 2026 | SKU Accuracy table: accuracy bucket stat chips (Total / >90% / 80–90% / 60–80% / <60%) above the table; each chip shows count and doubles as a filter button (`_skuBucketFilter` + `setSkuBucketFilter`); bucket counts always reflect Self/Comp scope before bucket filter is applied |
| June 2026 | SKU Coverage card (`#skuCoverageCard`, rendered by `renderSkuCoverageCard()`): masterdata SKU universe vs. GT SKUs seen in accuracy data, split All / Self / Competitor; animated progress bars color-coded by coverage tier; warning banner when masterdata SKUs have no GT observations; uses `APP.d.rows` (unfiltered) so sidebar filters don't deflate coverage numbers |
| June 2026 | HANDOFF.md updated for session 7 |
| July 2026 | **Session 8:** Fixed duplicate rows in SKU accuracy table — `buildSkuAccuracy` now keys accumulator by `dispName.toLowerCase()` (canonical display_name from masterdata) instead of `r.gt.toLowerCase()`; resolves the double-indexing in `mMap` (class_name + display_name both present as keys) |
| July 2026 | Removed 6-image cap in `buildDrillContent` — `.slice(0, 6)` dropped; all affected images per predicted SKU now shown; column header updated |
| July 2026 | Fixed SKU drill-down self-error: added `predSku.toLowerCase() === key` canonical match check alongside `r.is_correct`; rows where GT and pred differ only in string form but resolve to same masterdata display_name are now counted as correct, not errors |
| July 2026 | HANDOFF.md updated for session 8 |

---

## Section 12: Context Recovery Prompt

```
I'm working on a single-file HTML browser app at:
  /Users/kartik/Desktop/Accuracy Dashboard/index.html

Deployed at: https://kartik-15.github.io/cv-accuracy-dashboard/
GitHub repo: https://github.com/Kartik-15/cv-accuracy-dashboard

PROJECT: CV Accuracy Dashboard for analyzing computer vision model accuracy on NARTD cooler
shelf bounding-box predictions. 100% client-side, no server. Brand-agnostic (labels use
"Self"/"Competitor", not "CCI"/"competitor").

STACK: Vanilla JS + Tailwind CDN + Chart.js + PapaParse + SheetJS. No build tools.

INPUTS (multi-model detection + shared masterdata + optional IQ):
  1. Detection Data — one or more files (XLSX/CSV), one per model version:
     Required cols: session_id, QC_Display Name (GT), Display Name (Pred), competition
     Optional: file_path (GCS URL for image chips), quality (embedded fallback), date (for trend tracking)
     Each slot has a model name text input (user-typed). Date auto-read from 'date' column.
  2. Image Quality (CSV) — OPTIONAL: cooler_session_id → image_quality
  3. Masterdata (CSV, CGC pivot format): class_name, attribute_name, attribute_value,
     group_name (DIRECT COLUMN — not an attribute pair), competition
     IMPORTANT: class_name must match GT/pred display names exactly (case-insensitive).
     For Warm Shelves project, use New/WarmShelves_Masterdata.csv — the standard
     groupclassinfo CSV has wrong class_names for that project.

KEY RULES:
  - is_correct = (QC_Display Name === Display Name) — always recomputed, never from file
  - ACCURACY LEVELS: brand = pred_brand===gt_brand; variant = brand+variant match; SKU = is_correct
  - competition column in Raw = PREDICTION's type; GT competition from masterdata only
  - Masterdata keys: displayName.toLowerCase()
  - group_name is a DIRECT CSV COLUMN in CGC masterdata — read as row['group_name'] in
    buildMasterdataMap, stored as attrs._gname, mapped to entry.group
  - Enriched rows include: gt_group (from group_name direct col), gt_class (class_name pivot)
  - Quality normalization: "Good Quality"→"Good", "Blurred image"→"Blurred", etc.
  - Brand Demo: GT 'self' items only; top-15 competitor columns by error frequency (dynamic)
  - Brand Demo Self Misclass = r.comp==='self' && gt_brand !== pred_brand (cross-self brand error)
    Same-brand-different-SKU → Others (NOT Self Misclass)
  - Size Demo: same-brand, different-measure errors only (both measures non-zero)
  - KPI thresholds: ≥90%=green, 75-89%=amber, <75%=red
  - Drill-downs aggregate by brand name via getPredBrandLabel(r), not Display Name
  - Brand tab: buildBrandDrillContent(brandName, errors, total, colspan) — counts only, no inline chips
    toggleBrandErrorImg(brandName, errorBrand) → APP._brandErrorImgExpanded["B||E"] toggle
  - Variant/SKU tabs: buildDrillContent(errors, total, colspan) — 3 args, inline chips (unchanged)
  - Trend snapshots saved to localStorage key 'cci_accuracy_history_v1' after each processData()
  - Lazy tab rendering: renderActiveTabContent() called on switchMainTab()
  - Chart instances c1–c8 + _breakdownCharts; all destroyed in resetAll()
  - c1/c2/c3 are DEAD CODE (overview/brand chart removed) but resetAll still destroys safely
  - _isBlankVal(v): universal empty-value check
  - Breakdown charts: at BOTTOM of Brand Analysis tab; only when filter checkboxes checked;
    shared globalOrder (top-15 by combined volume) for all charts in a group

APP STATE (key additions since session 7):
APP = {
  files: { dets:[{file,modelName}], iq, md },  // ← dets is now an array
  d: { rows, qMap, mMap, ..., modelNames:[], dateRange:'' },
  _activeModel: 'all',          // model filter pill selection
  _trendsAccLevel: 'brand',     // 'brand'|'variant'|'sku'
  _trendsFilter: 'all',
  _trendsCompChart: null,       // comparison chart instance
  _variantAccuracyFilter: 'all',
  _variantAttrKey: '',          // selected attribute for Variant tab
  _variantGroupBy: 'brand+attr',
  _variantErrorImgExpanded: {},
  _variantBreakdownCharts: {},
  _skuAccuracyFilter: 'all',    // ← NEW session 7: 'all'|'self'|'competitor'
  _skuBucketFilter: 'all',      // ← NEW session 7: 'all'|'gt90'|'80to90'|'60to80'|'lt60'
  // ... rest unchanged from session 5
}

ENRICHED ROW NEW FIELDS (session 6):
  _model:    string  // model name from upload slot
  _fileDate: string  // YYYY-MM-DD from date column

SKU TAB LAYOUT (session 7):
  #skuCoverageCard  ← rendered by renderSkuCoverageCard(); uses APP.d.rows (unfiltered)
  #skuFilterBar     ← Self/Comp/All toggle (renderFilterToggle → setSkuAccuracyFilter)
  #skuStatChips     ← bucket chips rendered inline in renderSkuAccuracy()
  #skuSearch        ← text search
  skuHead/skuBody   ← SKU accuracy table; drill-down groups by r.pred (SKU-level, NOT brand)
  Size Demo card

SKU DRILL-DOWN (session 7):
  buildSkuAccuracy groups errors by r.pred (predicted Display Name)
  buildDrillContent called with label='SKU' → header = "Predicted SKU"
  (Variant tab still calls buildDrillContent without label → defaults to 'Brand')

MAIN_TABS = ['brand','variant','sku','trends']
FILTER_SECTIONS_KEY = 'cci_filter_sections_v2'
THEME_KEY = 'cci_theme'
HISTORY_KEY = 'cci_accuracy_history_v1'

HEADLINE STATS (5 cards): Images | Unique Brands | Unique Groups | Unique Classes | Unique SKUs
topStats bar now shows: N models · M dates · image count (no model ID / date fields)

CURRENT STATUS: SKU Analysis tab fully overhauled (session 7). Multi-model upload + Variant Analysis + IQ optional all live.
Next: SKU tab export CSV, verify multi-model flow end-to-end, MSL feature.

SAMPLE DATA FILES in same folder:
  - Accuracy Sheet.xlsx (detection data — use "Raw" sheet)
  - CCI_Image_Quality_Analysis - 12th June - Sheet1.csv (quality)
  - CGC_New_15th June.csv (CANONICAL masterdata — has group_name as direct column)
```
