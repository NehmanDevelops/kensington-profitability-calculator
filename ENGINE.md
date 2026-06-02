# Kensington Profitability Calculator — Engine Reference

This is the authoritative reference for the math behind the calculator. Every formula
below is documented and **validated to the dollar** against the "Henry Schein Revised"
worked example ($1.5M spend → Year-1 Net Profit = $12,036).

The calculator implements the full corporate profitability methodology — transaction
fees, commissions, back-end overrides, servicing costs and overhead — in one transparent
JavaScript function. The airline percentages the model relies on are surfaced directly as
editable inputs in the "Airline mix" table.

---

## 1. The two scenarios

| Toggle | Effect |
|---|---|
| **Does client book Air Canada contract rates?** | When **Yes** → Air Canada's non-commissionable share is forced to **100%**, so AC earns **no commission** (you don't earn commission on contracted fares). When **No** → AC commission flows normally. |

The calculator renders **two P&L columns side by side**: *Without AC Contract* and
*With AC Contract*. Everything else is identical between them.

---

## 2. Inputs

| Field | Henry Schein |
|---|---|
| Total spend | 1,500,000 |
| Provides total spend data? / unit | No / % |
| Provides avg spend/booking? | No |
| Books AC contract rates? | No |
| Needs OBT? | Yes |
| Hotel-only & Car-only bookings? | No |
| Non-GDS add-on % | 0% |
| Merchant of record? | Yes |
| Change "Current Booking Fee"? | No |
| Change "Other Fee"? | No |
| Spend share % (Air-Dom/Intl/Hotel/Car) | 46.53 / 24.27 / 26.83 / 2.37 |
| Spend share % **by BD** | 45 / 25 / 27 / 3 |
| Spend share $ **by BD** | 300k / 300k / 100k / 800k |
| Industry avg spend/bkg | 572.65 / 1661.55 / 511.28 / 222.47 |
| Client avg spend/bkg | 500 / 1500 / 600 / 200 |
| % managed: agent / online / online-assisted | 50 / 50 / 20 (all rows) |
| Airline total-sales % (Dom, Intl) | see calculator defaults |
| Airline **non-commissionable** % (Dom, Intl) | see defaults |
| Airline yield % (Dom, Intl) | see defaults |
| Hotel / Car commission yield | 8.70% / 5.56% |
| Hotel / Car commissionable % | 80% / 80% |
| Fee schedule (agent, online) standard | air 35/15, nonGDS 10/0, OA 15/0, hotel-only 20/15, car-only 20/15 |
| Fee schedule **by BD** | 40/15, 20/5, 20/20, 15/5, 15/5 |
| Other fees: applied?, standard, by BD | impl 500/250, s2go-imp 0/500, s2go-pnr 0/0.50, acct-mgmt 0/1500, OBT 500/500 |

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
(Categories: Air-Domestic, Air-International, Hotel, Car, plus Hotel-only and Car-only.)

---

## 3. Revenue components (per year, scaled by spending% `[0.80, 1.00, 1.00]`)

```
Commission Revenue:
  airDomComm = Σ_airlines  airDomBase * salesDom_a  * (1-nonCommDom_a) * yieldDom_a
  airIntlComm= Σ_airlines  airIntlBase* salesIntl_a * (1-nonCommIntl_a)* yieldIntl_a
      → if AC contract: nonCommDom_AC = nonCommIntl_AC = 1  (AC commission = 0)
  hotelComm  = base_hotel * hotelYield * hotelCommissionable%
  carComm    = base_car   * carYield   * carCommissionable%
  commrev    = airDomComm + airIntlComm + hotelComm + carComm

Service Fee Revenue: per category & channel
  feeAgent  = changeBookingFee=="No" ? stdAgentFee_c  : bdAgentFee_c
  feeOnline = changeBookingFee=="No" ? stdOnlineFee_c : bdOnlineFee_c
  rev_c = agent_c*feeAgent + online_c*feeOnline + onlineAssist_c*(feeOnline + onlineAssistFee)
  srvfeerev = Σ over Air-Dom, Air-Intl, Hotel-only, Car-only

Non-GDS Fee Revenue:
  = (totalAir+totalHotelCarOnly bookings) * nonGdsPct * nonGdsFee

Override Revenue:    [back-end airline/supplier overrides]
  = airBase * overrideAirPct  +  base_hotel*overrideHotelPct + base_car*overrideCarPct
  (override %: air 1.5%, hotel 0%, car 2%)

GDS Override Revenue:
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
Outcome ≥ target → green; below → gold/amber.

---

## 6. Validation (Henry Schein, Year 1 @ 80% spend, No AC contract)

| Line | Calculator | Reference model |
|---|---|---|
| Total Sales | 1,231,488 | 1,231,488 ✓ |
| Total Revenue | 88,306 | 88,306 ✓ |
| Total Direct Expenses | (28,242) | (28,242) ✓ |
| Gross Profit | 60,064 | 60,064 ✓ |
| Total Overhead | (48,028) | (48,028) ✓ |
| **Net Profit** | **12,036** | **12,036** ✓ |
| Growth Margin (incl ovr) | 7.17% | 7.17% ✓ |
| Net Margin (excl fixed) | 4.88% | 4.88% ✓ |
