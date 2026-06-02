# Maintenance Guide — Kensington Profitability Calculator

**For whoever maintains this after the original author leaves.** No build tools, no
server, no database. The whole calculator is one file you can edit in Notepad.

## What the files are

| File | What it is |
|---|---|
| `kensington-profitability-calculator.html` | **The app.** Open it in any browser. Self-contained — HTML + CSS + JavaScript in one file. |
| `ENGINE.md` | Plain-English reference for every formula in the model. Read this if you need to understand *why* a number is what it is. |
| `MAINTENANCE.md` | This file. |
| `.preview-server.ps1`, `.claude/` | Dev-only helpers for previewing inside the editor. **Not needed to run the app** — you can delete them. |

## How to run it

Double-click `kensington-profitability-calculator.html`. That's it. It runs entirely in
the browser. To share it, email the file or drop it on any web host (it's just static HTML).

## How to change things

Open `kensington-profitability-calculator.html` in a text editor. There are two clearly
labelled blocks near the bottom, inside `<script>`:

### 1. `CONFIG` — company-wide cost assumptions
These are the only numbers **not** shown on the form (standard company cost
assumptions). Each line has a comment. Examples:

```js
staffCostPerTxn:     16.66,   // sales-team cost per agent / online-assisted transaction
merchantFeePct:      0.032,   // merchant processing fee
overheadMarketingPct:0.012,   // overhead: marketing (% of total sales)
```
To change a rate, edit the number and save. Percentages are decimals (`0.032` = 3.2%).
Targets for the green/gold margin pills are under `target:`.

### 2. `DEF` — default values that pre-fill the form
These populate the form when it opens (currently the "Henry Schein" example). The BD can
type over any of them on screen; editing `DEF` just changes the starting point.

## How the two scenarios work

The right side shows two columns: **Without AC Contract** and **With AC Contract**. The
only difference between them: with the contract, Air Canada's commission is set to $0
(you don't earn commission on contracted fares). This is controlled in the `compute()`
function by the line:
```js
if(acContract && a.name==="Air Canada"){ ncD=1; ncI=1; }
```

## To re-skin / re-brand

Edit the 6 colour variables at the very top of the file under `:root{ ... }`
(`--brand`, `--gold`, etc.) and the `<h1>`/logo line in the `<header>`.

## Sanity check after any change

Open the file, expand the form sections, and confirm the **Henry Schein** defaults still
produce **Year-1 Net Profit ≈ $12,045** (no contract). That figure reconciles to the
reference financial model ($12,036; the few-dollar gap is rounding of the on-screen
share %). If it's wildly off, you changed something in `CONFIG` — undo it.

## To refresh the cost assumptions

If Finance updates the standard rates, `ENGINE.md` documents which `CONFIG` value drives
which line of the model, so you can refresh the constants in one place.
