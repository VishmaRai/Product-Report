# Daily Report Dashboard — Documentation

A single-file, mobile-friendly web app for reading a daily distribution report (Excel) and turning it into something you can actually scan: per-product sales, stock, branch targets, sales categories, and stock-health alerts.

The entire app is one file — `report-dashboard.html` — with no build step and no server. Open it in a browser, upload the day's report, and explore.

---

## 1. What it does

The raw report is ~500 products across 33 columns, which is unreadable at a glance. This app lets you:

- **Search** any product and see it on its own.
- See **per-branch sales and stock** for that product, instead of scanning a wide grid.
- Sort products into **sales categories** (Hero / Core / Slow / Dead) that adjust for how far into the month you are.
- Flag **branches underperforming their area target** (and tell you whether to push sales or send stock).
- Flag **overstocked and low-stock** products by how long the stock will last.
- Suggest **internal stock transfers** when one branch is dry while stock sits elsewhere.

---

## 2. The report file it expects

The app is built around one fixed layout and will keep working as long as that layout never changes.

- **File type:** `.xlsx` (also accepts `.xls`).
- **Filename:** the Bikram Sambat (BS) date, e.g. `2083-02-25.xlsx`. The app reads the date from the filename. If the filename has no date, it falls back to a date printed in the top rows of the sheet.
- **Sheets:** the first sheet is read (the workbook has an *Alphabetical* and a *Descending* sheet; either works since the app sorts internally).
- **Layout:** each row is one product. The sheet is split into two halves side by side — **sales on the left, closing stock on the right.**
- **Sales figures are month-to-date** (cumulative from day 1 of the BS month through the file's date), *not* single-day.

### Branches

Ten selling branches plus one central godown:

`KTM, NGD, BTL, POK, DMK, BTM, ITH, NPJ, LHN, ATR` — and **ONT** (godown), which appears only on the stock side because it doesn't sell, only supplies.

### Column positions (0-based, for maintenance)

| Field | Column |
|---|---|
| Product name | 1 |
| Company code | 2 |
| Sales per branch (KTM…ATR) | 3–12 |
| Total sold | 13 |
| Cost price (CP) | 14 |
| Sales amount | 15 |
| ONT godown stock | 19 |
| Stock per branch (KTM…ATR) | 20–29 |
| Total stock | 30 |
| Stock amount | 32 |

If the report layout ever shifts, these are the numbers to update (the `C` object near the top of the script).

---

## 3. Using the app

### Upload
On first open you get an upload screen. Choose the day's `.xlsx`. The app parses every product and takes you to the product list. Your last report is remembered (see *Persistence*), so on later opens it goes straight to the list.

### The product list
- **Search box** — type any part of a product name or company code.
- **Sales / Stock toggle** — switches the whole screen between two lenses:
  - **Sales lens:** category chips (All / Hero / Core / Slow / Dead), sorted by sales amount.
  - **Stock lens:** health chips (All / Overstock / Low / Healthy), sorted so the most frozen money is at the top.
- **Filter chips** — tap one to see *all* products in that group. The number on each chip is the exact count, and the list shows every one of them.
- Tap any product to open its detail screen.

### The product detail screen
- **Name, category badge, CP, company, ★ Star marker.**
- **Total sold** — quantity and rupee value. Mid-month it also shows **"on pace for Rs X"** (see *Projection*).
- **Closing stock** — quantity, rupee value, and a coloured **stock-health line** (e.g. "● Overstock · 4.2 mo cover").
- **Transfer suggestion** — a blue strip appears *only* when a branch is dry and still selling while stock is available elsewhere, e.g. "Send stock to DMK, ATR — ONT godown has 2,200."
- **By branch** — a card per branch showing Sold, Stock, the branch's target, and a flag where relevant (see *Branch targets*). The ONT godown card shows its stock as the supply source.

---

## 4. How the numbers work

### Month fraction (date awareness)
Because sales are month-to-date, the app works out how far through the month the file is:

> **f = day of month ÷ days in that BS month**

Example: `2083-02-25` falls in a 31-day month, so f = 25 ÷ 31 ≈ 0.81 (about 81% of the month elapsed).

BS months vary from 29 to 32 days, so the app carries a built-in **Bikram Sambat calendar table for years 2078–2091**. If a file's year ever falls outside that range, it falls back to a 30-day assumption.

### Sales categories
Categories are based on **month-to-date sales value**, with the thresholds scaled by `f` so a part-month file is judged on pace, not on raw totals:

| Category | Full-month threshold | Meaning |
|---|---|---|
| **Hero** | sales > Rs 100,000 × f | top sellers, on pace for 100K+ |
| **Core** | sales > Rs 20,000 × f | solid middle |
| **Slow** | sales > Rs 5,000 × f | weak |
| **Dead** | sales ≤ Rs 5,000 × f | barely moving |

At day 25/31 (f ≈ 0.81), the Hero line sits at about Rs 80,600 of month-to-date sales, and so on.

### Projection ("on pace for")
Shown only on mid-month files. It extrapolates the current pace to a full month:

> **projected = month-to-date sales ÷ f**

So a product at Rs 8,100 on day 25 of 31 is "on pace for ~Rs 10,000." On a last-day file there's nothing to project, so it's hidden.

### Branch targets (area weights)
Each branch is expected to take a fixed share of every product's total sales, based on its area:

| KTM | NGD | BTL | POK | DMK | BTM | ITH | NPJ | LHN | ATR |
|---|---|---|---|---|---|---|---|---|---|
| 25% | 15% | 11% | 20% | 4% | 7% | 10% | 3% | 2.5% | 2.5% |

For each product, a branch's **fair-share target = its % × the product's total sold.** A branch is flagged when it sells **less than 75%** of that fair share. Two flag types:

- **Push sales** (amber) — below target but the branch *has* stock: a selling problem to fix at the branch.
- **Send stock** (blue) — below target and the branch is *dry*: a supply problem to fix from the centre.

Branches at or above 75% stay plain, so only the gaps draw the eye. Targets below 1 unit are ignored (too small to judge meaningfully). Note: because both the target and the actual are month-to-date, the date cancels out here — no proration is needed for branch targets.

### Stock health (over / under stock)
Based on **months of cover = stock ÷ monthly selling rate**, where the monthly rate is the projected full-month sales (`month-to-date ÷ f`):

| Status | Rule | Meaning |
|---|---|---|
| **Overstock** | stock > 2 months of cover | money frozen on the shelf |
| **Low** | stock < ½ month of cover | about to run out / out of stock |
| **Healthy** | in between | fine |

A product selling with zero stock counts as **Low** (the most urgent kind). A product holding stock but selling nothing counts as **Overstock** (frozen). In the Stock lens, lists are ranked by **frozen rupee value** so the biggest money leads — except the **Low** filter, which ranks by sales at risk.

### Transfer suggestions
A transfer is suggested when a product has at least one branch that is **dry and still selling**. The suggested source is the **ONT godown** if it holds stock, otherwise the branch holding the most. It's a single line, shown only when there's an actual move to make.

---

## 5. Installing on your phone

The app behaves like an installed app via "Add to Home Screen":

1. Open `report-dashboard.html` in your phone's browser (Chrome).
2. Open the browser menu (⋮) → **Add to Home Screen**.
3. Launch from the new icon.

It ships with its own green chart icon and the name **"Daily Report,"** and includes a web-app manifest so it opens full-screen where supported. (A fully chrome-less launch is most reliable when the page is served over HTTPS rather than opened as a local file.)

---

## 6. Persistence and offline

The app remembers the **last report you loaded** using the browser's local storage, so reopening it brings up that report without re-uploading. "New file" replaces it; "Keep current report" backs out.

- This memory works when the file is opened normally on your phone or computer.
- It does **not** persist inside a sandboxed preview window — there it will ask for the file each time. This is expected and doesn't break anything.

---

## 7. Privacy

The report is treated as confidential. The app is a **viewer only** — when you upload a file, it is read **in your browser and never sent to any server.** This is why the chosen deployment is "host the public app code, upload the private file yourself": the application can live on any host, but the data never leaves your device.

Avoid putting the report file into a public GitHub repository or any URL without a login — anything a browser can fetch without authentication can be fetched by anyone. If automatic loading of the report is ever wanted, it should sit behind a real login wall (e.g. Cloudflare Access), not behind an obscure filename.

---

## 8. Limitations

- **Single file only.** The app reads one report at a time. There is no history or trend comparison yet.
- **Rate is estimated from one month.** "Months of cover," stock health, and the projection all assume this month's pace is representative. A seasonal or unusual month will read misleadingly. This is exactly what a future multi-file version improves, by averaging real history.
- **Fixed layout assumption.** The app reads columns by position. If the report's structure changes, the column map must be updated.

---

## 9. Roadmap

- **Multi-file trends** — load several daily files and track a product's stock building up or sales accelerating across the month. The dated reports and current logic are the building blocks for this.
- **Adjustable thresholds in-app** — expose the overstock/low cutoffs and the 75% target line as settings rather than fixed values.

---

## 10. Maintenance quick-reference

All tunable values live near the top of the `<script>` block in `report-dashboard.html`:

- `BS_CAL` — Bikram Sambat month lengths per year. Add new years here as needed.
- `TGT` — branch area-weight percentages.
- Category thresholds — inside the `categorize()` function (100000 / 20000 / 5000).
- `OVER` / `UNDER` — stock-health cutoffs (2 months / 0.5 month of cover).
- `HIT` — branch target hit ratio (0.75).
- `FLOOR` — smallest branch target worth flagging (1 unit).
- `C` — the column-position map for the Excel layout.
- `SALES_BRANCHES` — branch codes and their order.

No libraries to install; the only external dependency is SheetJS, loaded from a CDN for Excel parsing.
