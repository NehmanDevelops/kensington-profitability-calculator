# Kensington Profitability Calculator

A single-file, browser-based profitability calculator for Kensington Corporate travel deals.
It models a 3-year P&L for a client and compares two scenarios side by side:

- **Without an Air Canada contract** — Air Canada commission flows normally.
- **With an Air Canada contract** — Air Canada commission is $0 (you don't earn commission on contracted fares).

The engine was reverse-engineered from, and validated to the dollar against, the
*Corporate Profitability Model – TMC v3* Excel workbook.

## Run it
Open `index.html` in any browser. No build step, no server, no dependencies.

## Deploy (Vercel)
This is a static site. On Vercel, import this repo and use the **"Other"** framework preset
(no build command, output directory = root). `index.html` is served automatically.

## Maintain it
- `MAINTENANCE.md` — how to change rates, defaults, and branding (for non-developers).
- `ENGINE.md` — every formula, traced back to the original Excel cells.

All company cost assumptions live in the `CONFIG` block near the bottom of `index.html`.
