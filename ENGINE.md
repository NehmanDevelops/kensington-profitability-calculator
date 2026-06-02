# Profitability Engine — extracted from *Corporate Profitability Model – TMC v3.xlsx*

This is the authoritative reference for the math behind the Kensington Profitability
Calculator. Every formula below was reverse-engineered from the Excel workbook and
**validated to the dollar** against the "Henry Schein Revised" example
($1.5M spend → Net Profit Yr1 = $12,036).

The Excel hides the engine across 12 sheets (2 visible, 10 hidden) and pulls airline
percentages from pivot tables. The web calculator collapses that into one transparent
JavaScript function — the pivot-derived percentages simply become editable inputs in
the "Airline mix" table (which is exactly how the BD form already presents them).

---

## 1. The two scenarios

| Toggle | Excel cell | Effect |
|---|---|---|
| **Does client book Air Canada contract rates?** | `Sales Team Data!B10` | When **Yes** → Air Canada non-commissionable % is forced to **100%** (`D46=1`, `E46=1`), so AC earns **no commission**. When **No** → AC commission flows normally. |

The calculator renders **two P&L columns side by side**: *Without AC Contract* and
*With AC Contract*. Everything else is identical between them.

---

## 2. Inputs (Sales Team Data)

| Field | Cell | Henry Schein |
|---|---|---|
| Total spend | `B7` (`sales`) | 1,500,000 |
| Provides total spend data? | `B8` / unit `C8` | No / % |
| Provides avg spend/booking? | `B9` | No |
| Books AC contract rates? | `B10` | No |
| Needs OBT? | `B11` | Yes |
| Hotel-only & Car-only bookings? | `B12` | No |
| Non-GDS add-on % | `B13` | 0% |
| Merchant of record? | `B16` | Yes |
| Change "Current Booking Fee"? | `B17` | No |
| Change "Other Fee"? | `B18` (`otherfeechg`) | No |
| Spend share % (Air-Dom/Intl/Hotel/Car) | `B22:B25` | 46.53 / 24.27 / 26.83 / 2.37 |
| Spend share % **by BD** | `C22:C25` | 45 / 25 / 27 / 3 |
| Spend share $ **by BD** | `D22:D25` | 300k / 300k / 100k / 800k |
| Industry avg spend/bkg | `B28:B31` | 572.65 / 1661.55 / 511.28 / 222.47 |
| Client avg spend/bkg | `C28:C31` | 500 / 1500 / 600 / 200 |
| % managed: agent / online / online-assisted | `B35:D38` | 50 / 50 / 20 (all rows) |
| Airline total-sales % (Dom, Intl) | `B46:C52` | see calculator defaults |
| Airline **non-commissionable** % (Dom, Intl) | `D46:E52` | see defaults |
| Airline yield % (Dom, Intl) | `B63:C69` | see defaults |
| Hotel / Car commission yield | `B58 / B59` | 8.70% / 5.56% |
| Hotel / Car commissionable % | `C58 / C59` | 80% / 80% |
| Fee schedule (agent, online) standard | `B74:C79` | air 35/15, nonGDS 10/0, OA 15/0, hotel-only 20/15, car-only 20/15 |
| Fee schedule **by BD** | `D74:E79` | 40/15, 20/5, 20/20, 15/5, 15/5 |
| Other fees: applied?, standard, by BD | `B83:D87` | impl 500/250, s2go-imp 0/500, s2go-pnr 0/0.50, acct-mgmt 0/1500, OBT 500/500 |

**Booking-count base** per category `c`:
```
base_c = providesSpendData=="Yes"
           ? (unit=="%" ? totalSpend*shareBD_c : shareBD$_c)
           : totalSpend*share_c
avg_c  = providesAvgSpend=="No" ? industryAvg_c : clientAvg_c
agent_c        = ROUND(base_c/avg_c * agentPct_c, 0)
online_c       = needsOBT=="No" ? 0 : ROUND(base_c/avg_c * onlinePct_c*(1-assistPct_c), 0)
onlineAssist_c = needsOBT=="No" ? 0 : ROUND(base_c/avg_c * onlinePct_c*assistPct_c, 0)
total_c        = agent_c + online_c + onlineAssist_c
```
(Categories: Air-Dom row 8, Air-Intl row 9, Hotel row 10, Car row 11, +Hotel-only row 12, Car-only row 13.)

---

## 3. Revenue components (per year, scaled by spending% `[0.80, 1.00, 1.00]`)

```
Commission Revenue (commrev = K22):
  airDomComm = Σ_airlines  airDomBase * salesDom_a  * (1-nonCommDom_a) * yieldDom_a
  airIntlComm= Σ_airlines  airIntlBase* salesIntl_a * (1-nonCommIntl_a)* yieldIntl_a
      → if AC contract: nonCommDom_AC = nonCommIntl_AC = 1  (AC commission = 0)
  hotelComm  = base_hotel * hotelYield * hotelCommissionable%
  carComm    = base_car   * carYield   * carCommissionable%
  commrev    = airDomComm + airIntlComm + hotelComm + carComm

Service Fee Revenue (srvfeerev = N29): per category & channel
  feeAgent  = changeBookingFee=="No" ? stdAgentFee_c  : bdAgentFee_c
  feeOnline = changeBookingFee=="No" ? stdOnlineFee_c : bdOnlineFee_c
  rev_c = agent_c*feeAgent + online_c*feeOnline + onlineAssist_c*(feeOnline + onlineAssistFee)
  srvfeerev = Σ over Air-Dom, Air-Intl, Hotel-only, Car-only

Non-GDS Fee Revenue (nongdsrev = N31):
  = (totalAir+totalHotelCarOnly bookings) * nonGdsPct * nonGdsFee

Override Revenue (overriderev = K37):    [back-end airline/supplier overrides]
  = airBase*(shareAirDom+shareAirIntl-normalized)*overrideAirPct
  + base_hotel*overrideHotelPct + base_car*overrideCarPct
  (override %: air 1.5%, hotel 0%, car 2%)

GDS Override Revenue (gdsoverriderev = K44):
  = (totalAirDomBkgs*gdsSegPerDom + totalAirIntlBkgs*gdsSegPerIntl) * gdsOverridePerSegment
  (2.5 segs/dom bkg, 5.5 segs/intl bkg, $1.65/segment)

One-time / recurring fees:
  Implementation, S2Go-Imp, OBT-Imp  → Year 1 only
  S2Go-per-PNR, Account-Mgmt          → recurring
  Each: changeOtherFee=="No" ? standardFee : (applied=="No" ? 0 : bdFee)
  OBT-Imp also gated by needsOBT=="Yes"

Total Revenue = commrev*sp + srvfeerev*sp + nongdsrev*sp
              + implFee + s2goImp + s2goPnr*sp + obtImp + acctMgmtFee
              + (overriderev+gdsoverriderev)*sp
```

---

## 4. Expenses & profit

```
Direct Expenses:
  Sales Team Cost   = staffRate * Σ(agent_c + onlineAssist_c)  over air + hotel/car-only   [$16.66/txn]
  OBT Cost          = needsOBT ? obtBookingCost * Σ online_c (air+hotel/car-only) : 0       [$3.50/bkg]
  OBT Impl Cost     = Year1 & needsOBT ? obtImplCost : 0                                    [$1,000]
  S2Go per-PNR Cost = s2goCostPnr * sp
  Account Mgmt Cost = acctMgmtDailyRate * (acctMgmtDays/2)                                  [$493.23/day]
  24/7 Cost         = cost247PerBkg * totalAllBookings * sp                                 [$5/bkg]
  Merchant Fee Cost = merchantOfRecord ? (sum of fee-revenue rows) * merchantPct : 0        [3.2%]

Account-mgmt days tier by spend:  ≥$1M → 14.5,  ≥$500k → 6,  else → 1.5

Gross Profit = Total Revenue − Direct Expenses

Overhead (% of Total Sales): Marketing 1.2% + Operating 1.8% + Back-office 0.9%

Net Profit = Gross Profit − Overhead

Total Sales (display) = Σ_c (totalSpend*share_c*sp) + total fee revenue
```

---

## 5. Outcome margins (vs Target)

```
Growth Margin % (excl overrides) = (TotalRevenue − overrideRev) / TotalSales   target 6.5%
Growth Margin % (incl overrides) =  TotalRevenue / TotalSales                  target 8.0%
Net Margin %    (excl fixed)     =  GrossProfit  / TotalSales                  target 5.75%
```
Outcome ≥ target → green; below → gold/amber (mirrors the Excel "Outcome" block).

---

## 6. Validation (Henry Schein, Year 1 @ 80% spend, No AC contract)

| Line | Engine | Excel |
|---|---|---|
| Total Sales | 1,231,488 | 1,231,488 ✓ |
| Total Revenue | 88,306 | 88,306 ✓ |
| Total Direct Expenses | (28,242) | (28,242) ✓ |
| Gross Profit | 60,064 | 60,064 ✓ |
| Total Overhead | (48,028) | (48,028) ✓ |
| **Net Profit** | **12,036** | **12,036** ✓ |
| Growth Margin (incl ovr) | 7.17% | 7.17% ✓ |
| Net Margin (excl fixed) | 4.88% | 4.88% ✓ |
