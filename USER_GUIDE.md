# CV Accuracy Dashboard — User Guide

**Live tool:** https://kartik-15.github.io/cv-accuracy-dashboard/

---

## What This Tool Does

The CV Accuracy Dashboard lets you upload computer vision model output for NARTD cooler shelf images and instantly see how accurately the model identified products. Everything runs in your browser — no data is uploaded to any server.

---

## Getting Started

### Step 1 — Open the tool
Go to https://kartik-15.github.io/cv-accuracy-dashboard/ in Chrome (recommended) or any modern browser.

### Step 2 — Upload three files

| Slot | What to upload | Format |
|------|---------------|--------|
| **Detection Data** | Bounding-box predictions vs ground truth | XLSX (use the `Raw` sheet) or CSV |
| **Image Quality** | Maps each image session ID to a quality tier | CSV |
| **Masterdata** | Product catalog with brand, variant, size attributes | CSV (CGC pivot format) |

Drag and drop files onto the zones, or click each zone to browse. A green tick appears when a file is staged.

### Step 3 — Enter model details *(optional but recommended)*
- **Model ID / Version** — a short label for this run, e.g. `v1.2` or `YOLOv8-epoch-50`. Used to track trends over time.
- **Run Date** — defaults to today.

### Step 4 — Process Data
Click **Process Data**. A column mapping dialog appears automatically.

### Step 5 — Confirm column mapping
The tool auto-detects which column in each file maps to which field. Review the dropdowns and correct any mismatches, then click **Confirm & Process**. Processing typically takes a few seconds.

---

## Dashboard Layout

Once processing completes, the dashboard has three main areas:

```
┌─────────────────────────────────────────────────────────┐
│  Top bar:  model ID · date · image count · theme toggle  │
├──────────┬──────────────────────────────────────────────┤
│ Filters  │  Headline stats → KPI cards → Tab content    │
│ sidebar  │                                              │
└──────────┴──────────────────────────────────────────────┘
```

### Headline Stats (always visible)
Four cards at the top show counts for the filtered data: **Images**, **Unique Brands**, **Unique Variants**, **Unique Groups**.

### KPI Cards
Four accuracy metrics computed across all filtered bounding boxes:

| Metric | Meaning |
|--------|---------|
| **Object Accuracy** | % of bounding boxes correctly identified |
| **Macro Precision** | Average precision across all GT classes |
| **Macro Recall** | Average recall across all GT classes |
| **F1-Score** | Harmonic mean of precision and recall |

Color coding: **green** ≥ 90%, **amber** 75–89%, **red** < 75%.

---

## Tabs

### Overview
Two summary charts:
- **Accuracy by Image Quality** — bar chart showing how accuracy varies across Good / Average / Poor / Blank / Blurred image tiers.
- **Brand vs Variant vs SKU Accuracy (Top 10)** — grouped bar chart comparing accuracy at three granularity levels for the top 10 brands by volume.

### Brand Analysis
- **Top 15 bar chart** — horizontal bars showing accuracy for the 15 highest-volume brands.
- **Brand Accuracy table** — every brand with correct count, error count, total bounding boxes, and accuracy %. CCI brands are labeled **CCI**; competitors labeled **Comp.**
- **Drill-down** — click any brand row to expand and see which competitor brands the model confused it with, error counts, % share of that brand's errors, and up to 6 clickable image links.
- **Brand Demonstration Sheet** — below the accuracy table; image-level breakdown showing, for each image, how many CCI bounding boxes were correct (Self Facings), same-brand wrong SKU (Self Misclass.), predicted as Non-NARTD, predicted as Others, and predicted as each of the top 15 competitor brands. Searchable, paginated, exportable to CSV.

### Variant Analysis
Same structure as Brand Analysis but grouped by Brand + Pack Type. Drill-downs show which predicted brands were responsible for errors.

### SKU Analysis
- **SKU Accuracy table** — every individual SKU (QC Display Name) sorted by most errors first.
- **Drill-down** — expand any SKU to see the brand-level breakdown of misclassifications and affected images.
- **Size Demonstration Sheet** — below the SKU table; image-level breakdown of same-brand, wrong-size errors (e.g., 330ml predicted as 500ml). Searchable, paginated, exportable to CSV.

### Trends
Tracks accuracy over multiple model runs. Each time you process a file, a snapshot is saved automatically in your browser.

- **Model Run History** — table of all saved runs. Click the trash icon to delete a run; **Clear All** removes everything.
- **Overall KPI Trends** — line chart of Accuracy, Precision, Recall, F1 across runs.
- **Brand / Variant / SKU Trend Charts** — use the selector pills to choose which brands/variants/SKUs to plot across runs.

> Trend data is stored in your browser's localStorage. It survives page refreshes but will be lost if you clear browser storage.

---

## Filters

The sidebar on the left controls what data is included in all charts and tables simultaneously.

### Choosing which filter types to show
The **toggle pills** at the top of the sidebar show every available filter dimension — one for each attribute in your masterdata (Brand, Pack type, Variant, Sub category, measure, etc.) plus Image Quality. Click a pill to show or hide that filter section. Your preference is saved and remembered across sessions.

### Applying filters
Each visible filter section lists all unique values for that attribute. Check any values to include only rows matching those values. Multiple selections within a section are OR (any match passes); across sections they are AND (all must pass).

**Reset All** clears all active filter selections (but keeps your section visibility preferences).

---

## Drill-Downs (Brand / Variant / SKU tabs)

Click any row in the Brand Accuracy, Variant Accuracy, or SKU Accuracy table to expand it. The drill-down shows:

| Column | What it means |
|--------|--------------|
| **Predicted Brand** | The brand the model predicted instead of the correct one |
| **Count** | Number of bounding boxes misclassified as that brand |
| **% of Errors** | Share of that item's total errors |
| **Affected Images** | Clickable chips linking directly to the image in ShelfWatch viewer |

Click an image chip (e.g. `abc123… · Good`) to open that image in the ShelfWatch viewer in a new tab.

---

## Brand Demonstration Sheet

Found at the bottom of the **Brand Analysis** tab.

Each row = one image (session ID). Columns:

| Column | Definition |
|--------|-----------|
| **Image ID** | Session identifier (click the image icon to view in ShelfWatch) |
| **Image Quality** | Quality tier of this image |
| **Self Facings** | CCI GT boxes correctly identified |
| **Self Misclassification** | CCI GT boxes predicted as same brand but wrong SKU |
| **Non-NARTD** | CCI GT boxes predicted as Non-NARTD |
| **Others** | CCI GT boxes predicted as Others or unknown brand |
| **[Brand X]** columns | CCI GT boxes incorrectly predicted as that competitor brand |
| **Total GT** | Total CCI GT bounding boxes in this image |

> The brand columns show **misclassifications only** — how many CCI products were wrongly predicted as that brand. The columns are the top 15 competitor brands by actual error frequency in the filtered data (not all brands in masterdata).

Click **Export** to download as CSV.

---

## Size Demonstration Sheet

Found at the bottom of the **SKU Analysis** tab.

Shows images where the model got the brand right but predicted the wrong size (e.g., Thums UP 330ml predicted as Thums UP 500ml). Only same-brand, different-measure errors appear here.

Each row = one image. Columns: Image ID, Image Quality, Total Size Errors, then one column per brand that has size errors in the filtered data.

Click **Export** to download as CSV.

---

## Sorting Tables

Every column header in the five accuracy and demo tables is **clickable**:
- First click → sort **descending** (highest first)
- Second click → sort **ascending**
- Click a different column → switch sort to that column

A ↕ icon (dim) shows on unsorted columns; ↑ or ↓ (blue) shows on the active sort column.

---

## Light / Dark Theme

Click the **sun/moon icon** (☀ / ☾) in the top-right corner of the top bar to switch between dark and light themes. Your preference is saved automatically.

---

## Exporting Data

Both the Brand Demo Sheet and Size Demo Sheet have an **Export** button that downloads the current filtered + paginated data (all pages, not just the current page) as a UTF-8 CSV file compatible with Excel.

---

## Frequently Asked Questions

**Q: Why does my data show 0% quality-matched?**
A: The session IDs in your Detection Data and Image Quality files don't match. Check that you're using the correct column in the column mapping dialog. Look for a yellow warning banner at the top of the dashboard.

**Q: Some brands are missing from the Brand Demo columns.**
A: The Brand Demo columns show only the top 15 competitor brands by actual error frequency in the currently filtered data. If a brand has no errors in the filtered view, it won't appear.

**Q: My trend charts are empty.**
A: Trends require at least two processed runs saved in your browser. Each time you click Process Data, a snapshot is saved automatically. Run the tool with a second model's data and the trend lines will appear.

**Q: The tool is stuck on the loading screen.**
A: This usually means a column couldn't be mapped correctly. Try refreshing the page and re-uploading, paying attention to the column mapping dialog before clicking Confirm.

**Q: Will my data be saved if I close the browser?**
A: The uploaded file data is **not** saved — you need to re-upload each session. However, your **trend history**, **filter section preferences**, and **theme choice** are all saved in browser localStorage and will persist across sessions.
