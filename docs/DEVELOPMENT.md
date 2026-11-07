# Development notes & changelog

> Detailed implementation history, build notes, and testing record for the Crummy Stock
> desktop app. See the main [README](../README.md) for the project overview.

Last updated: 2026-11-06

This document covers the file layout, release/dev builds, and all recent changes to the
PC web app.

---

## Folder layout

| Path | Purpose |
|---|---|
| `release\Stock App.html` | Main release app (source of truth) |
| `dev\Stock App.dev.html` | Generated dev build with sample data |
| `backup\` | Historical backup HTML snapshots |
| `resources\CrummyStock.lottie` | Original Lottie logo animation |
| `profiles\release.seed.json` | Empty default data for release |
| `profiles\dev.seed.json` | Sample default data for dev |
| `testData.zip` | Source backup used for the current dev sample data |
| `build-profiles.ps1` | Builds the release and dev variants |

The main app is a single self-contained HTML file. QR generation, ZIP backup support,
and the Lottie logo player are all embedded inline, so it works from the network share
without external files or a CDN.

Historical backups in `backup\`:

- `Stock App.backup-20261105-150350.html`
- `Stock App.backup-20261105-152706-with-sample-data.html`
- `Stock App.backup-20261105-154232-before-theme.html`

---

## Where the data lives

- Inventory, products, photos, categories, promos, tags, sales history, and settings
  are stored in this browser's IndexedDB on this computer.
- Release uses the `stock-app` database. Dev uses `stock-app-dev`.
- Nothing is sent to a server or cloud automatically.
- Clearing browser data, using a private window, moving or renaming the app file, or
  losing this machine can make the data inaccessible.
- The app requests persistent storage from the browser at startup, but that is not a
  substitute for backups.
- Use **⚙ Config → Backup → Export**, or the warning bar's **Export now** button,
  regularly. The warning bar also shows when the last backup was exported.

---

## Language

- Open **⚙ Config → Language** to choose **Automatic (browser)**, **English**, or
  **Español**.
- **Automatic** follows the browser language: a browser reporting `es` or `es-*`
  (for example `es-CL`) gets Spanish; everything else starts in English.
- The choice applies immediately, including the navigation, dialogs, warnings, sales
  history, and Sales Excel export.
- The selection is stored in this browser as `stock-lang` and survives reloads.
- Language is a desktop UI preference and is not included in inventory backups.

---

## Currency

- By default, **Automatic (from backup)** uses the phone backup's `currencyId` and
  `showMoneyDecimals` settings.
- Open **⚙ Config → Currency** to override the currency or decimals:
  - **Automatic (from backup)**, or USD, EUR, GBP, JPY, BRL, INR, or CHF.
  - **Automatic (from backup)**, **Show decimals**, or **Hide decimals**.
- When decimals are off, amounts are rounded to whole units like the mobile app; when
  on, two decimal places are shown.
- Changes apply to inventory values, product prices, promos, sales history, and Excel
  exports. Excel headers include the active currency symbol.
- Overrides are stored in this browser as `stock-currency` and `stock-decimals`.
  They do not convert prices and do not change the phone backup.
- The supported currency list matches the mobile app.

---

## Taxes and card fees

- **⚙ Config → Taxes & fees** edits the same settings the mobile app uses:
  - **General tax (%)**
  - **Card fee (%)**
  - **Fixed card fee** (uses the active currency symbol)
- Rates are stored back into the backup settings as basis points:
  `taxRateBasisPoints`, `cardFeeRateBasisPoints`, and `cardFeeFixedCents`.
- These values are sent to the phone in the backup and apply to new sales there.
- Changing them does not recalculate historical sales; existing orders keep the tax and
  card fee stored with them.
- The Sales summary shows totals for **Tax** and **Card fees** across the filtered
  orders.
- Card fees apply only to card payments on the phone; the percentage and fixed amount
  are added per transaction.

---

## Release and dev builds

Run these commands from the repository root:

```powershell
.\build-profiles.ps1 -Profile dev
.\build-profiles.ps1 -Profile release
.\build-profiles.ps1 -Profile release -ClearData
.\build-profiles.ps1 -Profile both
```

If PowerShell blocks the script:

```powershell
powershell -ExecutionPolicy Bypass -File .\build-profiles.ps1 -Profile dev
```

| Profile | Output | Default data | Database |
|---|---|---|---|
| release | `release\Stock App.html` | Empty inventory | `stock-app` |
| dev | `dev\Stock App.dev.html` | Test inventory | `stock-app-dev` |

### Current dev data

The dev seed (`profiles\dev.seed.json`) was built from `testData.zip`, with generated
sales history added for testing:

- 6 categories
- 40 products
- 40 product photos
- 45 orders
- 94 sale lines
- 4 older `legacySales` entries
- 39 distinct order dates (25 Aug 2026 to 6 Nov 2026)
- 16 orders with historical promo discounts
- 19 cash, 19 card, and 7 other-payment orders
- 0 current promos
- 0 tags

The backup was exported by phone app version 1.4.0, schema version 10, and includes the
newer `providers`, `materials`, `materialEntries`, and `materialEntryLines` fields.

### Important behaviour

- The release file itself contains no inventory. If you see products, they are saved
  in the browser's IndexedDB from earlier use.
- Release keeps using the existing `stock-app` database, so normal updates do not wipe
  a user's saved inventory.
- Dev uses a separate `stock-app-dev` database, so dev data never mixes with release
  data.
- The dev build shows a `DEV` badge in the header and `(dev)` in the browser tab title.
- `dev\Stock App.dev.html` is generated. Do not edit it directly; edit
  `release\Stock App.html` and rebuild.

### One-time clean release

`-ClearData` adds a one-time reset token to a release build. On the next load, the app:

1. Clears the saved release inventory.
2. Records the reset token in the `stock-app` database.
3. Starts with the empty release seed.

Later loads see the recorded token and keep data normally, so the reset does not happen
again. The marker is stored in IndexedDB rather than localStorage, so it also works when
localStorage is unavailable.

The current release build (`release\Stock App.html`) still carries its pending reset
token from the 2026-11-06 build. Opening it once will clear any previously saved release
inventory.

To clear data manually at any time: **⚙ Config → Backup → Clear all data**.

---

## Changes made

### 1. Fixed cross-tab event handler leaks

**Problem:** After visiting **Categories & tags**, then returning to **Inventory**, the
old categories handler was still attached to the shared `#main` element. Clicking
**+ Add product**, **+**, **−**, or **Edit** could therefore trigger the old
“Remove it?” category/tag prompt.

**Fix:**

- `render()` now clears delegated handlers on `#main` before switching views:
  `onclick`, `onchange`, `oninput`, `onkeydown`, and `onsubmit`.
- The Categories & tags handler now only reacts to buttons with
  `data-k="categories"` or `data-k="tags"`.
- The same fix clears the Promos and Theme view handlers when leaving those tabs.
- Inventory resets a stale category filter if that category no longer exists, for
  example after deleting it or importing another backup.

### 2. Fixed scanner cleanup

- Closing the scanner dialog with **Esc** now stops the camera instead of leaving it
  running in the background.
- A barcode detected after the dialog has closed can no longer switch the app back to
  Inventory unexpectedly.

### 3. Added promo editing

The **Promos** tab now supports full promo management on the PC:

- **+ New promo**
- **Edit**
- **Remove**
- Promo name
- Applies to a category or selected products
- Units-needed threshold
- Fixed-amount or percentage discount
- Optional start/end schedule
- Enabled toggle
- Status labels: Active, Scheduled, Expired, Off

Promo values are stored in the same format as the phone app:

- Fixed discounts in cents.
- Percent discounts in basis points (`1050` = `10.5%`).
- Dates as epoch milliseconds or `null`.
- Product IDs as a comma-separated string such as `"1,3"` (`Promo.productIdsCsv`).

Legacy `"[]"` product ID values are also handled.

### 4. Added the Config tab

The main navigation is now:

**Inventory · Sales · Categories & tags · Promos · Phone · ⚙ Config**

- **Theme**, **Language**, **Currency**, **Taxes & fees**, and **Backup** moved into
  Config.
- The gear tab is rightmost.
- Config remembers the last subsection used.
- Theme and Backup continue to work inside the Config shell without wiping its
  sub-tabs.

### 5. Added release/dev profiles

- Added the profile block in the app: `PROFILE`, `DBNAME`, `RESET_TOKEN`, and `SEED`.
- Added `profiles\release.seed.json` and `profiles\dev.seed.json`.
- Added `build-profiles.ps1`.
- Added separate databases for release and dev.
- Added the dev `DEV` badge.
- Added the one-time release reset mechanism with `-ClearData`.

### 6. Added the animated Lottie logo

- `resources\CrummyStock.lottie` is embedded as the header logo.
- Replaced the 🛒 emoji with the 600×600 animation at 56×56 px.
- Uses lottie-web 5.12.2 SVG player, embedded inline.
- All 12 animation images are converted to data URLs inside the HTML.
- The 🛒 emoji remains as a fallback if Lottie cannot start.
- Pauses when the browser tab is hidden.
- Respects `prefers-reduced-motion` and shows a still frame instead.

**Maintenance:** the Lottie data is embedded manually. If
`resources\CrummyStock.lottie` changes, the animation must be re-embedded into
`release\Stock App.html`, then the dev build regenerated.

### 7. Reorganised folders

Files were moved from the `StockerWeb` root into:

- `backup\` — old backup snapshots
- `release\` — release app
- `dev\` — dev app
- `resources\` — Lottie source

`build-profiles.ps1` and the `profiles\` folder stay in the root. The build script and
all paths were updated to match the new layout.

### 8. Replaced dev sample data with testData.zip

- `profiles\dev.seed.json` now contains the backup and photos from `testData.zip`.
- The dev build was regenerated: 6 categories, 40 products, and 40 photos.
- All 40 product image paths were verified against the embedded images.
- Release data and its pending reset token were left unchanged.

### 9. Added square photo cropper and white PNG backgrounds

- Choosing a product photo now opens a **Crop photo** dialog instead of saving the
  image immediately.
- The crop area is square and becomes a **560×560 JPEG product thumbnail**.
- The photo can be dragged to position it, zoomed with the slider or mouse wheel
  (100%–400%), and re-centred with **Center**.
- Transparent PNG areas are flattened onto **white**, both in the cropper and in the
  normal image resize path used by backup imports. They no longer turn black.
- Cancel leaves the existing product photo and preview unchanged.

### 10. Products can have no category

- The product editor now includes a **No category** option.
- Saving with **No category** stores `categoryId: null`.
- Products can be created and saved even when the app has no categories at all.
- Product cards show **No category** instead of a dash.
- The Inventory category filter has a **No category** option.
- Missing or deleted category IDs are treated as uncategorized. Opening such a
  product shows **No category**, and saving repairs it to `null`.
- Existing behaviour is preserved for new products when categories exist: the first
  category is selected by default, but **No category** can be chosen.

### 11. Added sales history explorer

- New **Sales** tab in the main navigation: **Inventory · Sales · Categories & tags ·
  Promos · Phone · ⚙ Config**.
- Reads the phone backup's `orders`, `sales` (or `saleLines`), and `legacySales`
  arrays.
- Summary cards show orders, revenue, items sold, and average order value.
- Orders are listed newest first and expand to show line items, item subtotal, promo
  discount, tax, card fee, total, and order note.
- Order lines show product name, barcode, quantity, unit price, and line total.
- Search matches order ID, product name, barcode, note, and promo name.
- Payment filter: All, Cash, Card, Other.
- Optional From/To date filters, plus a Clear button.
- Older `legacySales` entries are shown in an "Older sales" section.
- Orders missing a header are rebuilt from their saved sale lines, with a note that
  tax and fees were not stored.
- Backup JSON using either `sales` or `saleLines` is supported.
- Sales amounts are shown with two decimal places.

### 12. Added local-storage warning and backup reminder

- Confirmed that all app data — including photos and sales history — is saved in the
  browser's IndexedDB (`stock-app` release, `stock-app-dev` dev).
- Added a warning bar under the header: inventory, photos, and sales are local to this
  browser and computer, and clearing browser data or moving the app file can erase
  them.
- The warning bar includes **Backup options** and **Export now** buttons.
- It shows when the last backup was exported from this browser, and says when no
  backup has been made yet.
- Added a stronger warning and the same last-backup line to the Backup section.
- Exporting a `.zip` through the Backup section or the warning bar updates the
  last-backup time.
- The app now requests persistent browser storage at startup.
- If the browser blocks local storage, the warning bar says changes will be lost when
  the page closes.

### 13. Added Excel export for sales

- The Sales tab now has an **Export Excel** button.
- It exports the sales currently shown, so the search, payment, and date filters apply.
  Filtered exports get a `-filtered` suffix in the filename.
- The `.xlsx` workbook contains:
  - **Orders** — order ID, date, payment, item count, subtotal, promo discount, promo
    name, tax, card fee, total, and note.
  - **Order details** — one row per ordered product, with the order ID, date, payment,
    product, barcode, quantity, unit price, line total, order subtotal, promo, tax,
    card fee, order total, and note.
  - **Items** — order ID, date, product, barcode, quantity, unit price, and line total.
  - **Older sales** — included when `legacySales` entries are present.
- Dates are real Excel dates, money values are numeric with a currency format, and
  headers are bold, frozen, and filterable.
- The workbook is generated with the JSZip library already embedded in the app, so no
  extra dependency is needed.
- Promo discounts are written as negative amounts so totals read naturally.

### 14. Added dev sales history test data

- `profiles\dev.seed.json` now contains 45 generated orders with 94 sale lines.
- The orders span 39 distinct dates from 25 August 2026 to 6 November 2026, including
  several orders on the most recent days.
- Payment methods are mixed: 19 cash, 19 card, and 7 other.
- 16 orders include historical promo discounts, and some orders have notes.
- 4 `legacySales` entries from 100–200 days ago populate the Older sales section.
- Order subtotals, item counts, totals, product names, barcodes, and prices all match
  the dev products; counts in the backup were updated.
- The dev build was regenerated with the new sales data.

### 15. Fixed the Sales date filter losing focus

- The Sales filters no longer rebuild the whole filter bar when a value changes.
- Date inputs now update the summary and list in place, so typing a year does not
  deselect the input or drop focus.
- Search also updates results in place and keeps the caret position naturally.
- The Clear button resets the controls without replacing them.
- This was verified in a real browser by typing a full date into the From field.

### 16. Added Spanish translation, language detection and language menu

- The whole desktop UI is available in **English** and **Spanish**, including
  navigation, inventory, product/category/promo dialogs, sales history, phone transfer,
  config, theme, backup, cropper, scanner, warnings and toasts.
- **Automatic (browser)** is the default: `es`/`es-*` browser languages use Spanish,
  everything else uses English.
- Added **⚙ Config → Language** with Automatic, English, and Español.
- The selection applies immediately and is stored in `localStorage` as `stock-lang`, so
  it survives reloads.
- Sales dates use the selected locale. Excel export sheet names, column headers and
  filename are localized (`ventas-...xlsx` in Spanish).
- Built-in theme names are translated for display; user-created theme names are kept
  as entered.

### 17. Added mobile currency settings and Currency config

- The app now reads `settings.currencyId` and `settings.showMoneyDecimals` from the
  phone backup instead of hard-coding USD.
- Added **⚙ Config → Currency**:
  - Automatic (from backup) or a manual currency: USD, EUR, GBP, JPY, BRL, INR, CHF.
  - Automatic (from backup), Show decimals, or Hide decimals.
- Amounts now use the active currency symbol and match the mobile decimal behaviour:
  whole units when decimals are off, two decimals when on.
- Product price labels, promo fixed-amount labels, stock values, sales totals, and Excel
  headers all use the active currency symbol.
- Excel money columns use the selected decimal setting and display the currency in the
  column header.
- Manual overrides are stored as `stock-currency` and `stock-decimals`; they do not
  convert prices or change the phone backup.

### 18. Added mobile tax and card-fee settings

- Added **⚙ Config → Taxes & fees** with General tax (%), Card fee (%), and Fixed
  card fee inputs.
- Values are read from and written to the backup settings as
  `taxRateBasisPoints`, `cardFeeRateBasisPoints`, and `cardFeeFixedCents`.
- The fixed fee label uses the active currency symbol.
- Save validates rates between 0% and 100%.
- These settings travel to the phone in the backup and are used for new sales;
  historical orders keep their stored tax and card fees.
- The Sales summary now shows total **Tax** and **Card fees** for the filtered orders.
- The Backup summary line shows the fixed card fee alongside the tax and card-fee
  percentages.

---

## Testing performed

- JavaScript syntax checks on both builds.
- DOM tests with jsdom and fake IndexedDB.
- Verified the cross-tab handler leak is fixed.
- Verified promo create/edit/delete, scheduling, CSV serialization, and backup shape.
- Verified the Config tab and its Theme/Backup sections.
- Verified release reset clears old data once and keeps later saves.
- Verified dev uses its own database and loads the new 40-product test data.
- Rendered both builds in headless Edge to confirm the Lottie logo displays.
- Ran a real-browser crop test with a transparent PNG: verified zoom, drag, centring,
  560×560 output, white transparent corners, and the red subject in the centre.
- Verified the backup-import resize path also produces a white background for
  transparent PNGs.
- Verified products can be created, edited, filtered, and displayed with no category,
  including when the app has no categories at all.
- Verified sales stats, order details, line items, legacy sales, search, payment and
  date filters, Clear, empty state, and the `saleLines` fallback.
- Rendered the Sales view in headless Edge to check the summary cards, filters, order
  rows, and expanded line-item layout.
- Verified products, categories, and image data are written to IndexedDB and loaded
  back correctly after a fresh app start.
- Verified the local-data warning bar, Backup options action, Export now download,
  last-backup reminder, and reminder persistence after reload.
- Verified the Sales Excel export downloads a valid `.xlsx`; opened it with openpyxl
  and checked the Orders, Order details, Items, and Older sales sheets, dates, money
  values, the per-product order rows, and the filtered-export variant.
- Verified the dev sales test data: 45 orders, 94 lines, and 4 legacy sales are
  consistent, and the Sales view renders them across 39 distinct dates.
- Verified the Sales date filter keeps focus and selection while typing a full date,
  search keeps focus, and Clear resets the controls without replacing them.
- Verified Spanish auto-detection with an `es-CL` browser and the complete Spanish UI,
  including inventory, sales, categories, promos, phone, config, theme, backup,
  product/promo/get-transfer dialogs, scanner and cropper.
- Verified switching to English, persistence across reloads, and switching back to
  Automatic restores Spanish from the browser locale.
- Verified the Spanish Sales Excel export uses `Pedidos`, `Detalle de pedidos`,
  `Artículos`, and `Ventas anteriores`, with translated headers and `ventas-` filename.
- Verified automatic currency/decimal settings from a EUR backup, manual USD override,
  automatic fallback to the backup, whole-number rounding when decimals are off, and
  persistence in `stock-currency` / `stock-decimals`.
- Verified product labels, sales totals, Spanish currency labels, and Excel headers use
  the active currency symbol; opened the EUR workbook with openpyxl and confirmed
  `Subtotal (€)` headers and 2-decimal number formats.
- Verified tax and card-fee values load from the backup, save correctly, reject rates
  above 100%, appear in the Sales tax/card-fee totals, show the fixed fee in the Backup
  summary, and display translated labels in Spanish.
