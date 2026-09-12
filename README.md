# AI-Driven Supply Chain Forecast & Resilience Platform

A working implementation of the project described on your resume: an AI-enabled
control tower for a 2-echelon supply chain network that forecasts demand and
quantifies how the network holds up under disruption.

## 1. What was built

```
3 Suppliers  --->  1 Central Warehouse  --->  5 Retail Stores
(A, B, C)          (inventory buffer)         (Store_1 ... Store_5)
```

| Module | File | What it does |
|---|---|---|
| Data | `data/generate_data.py` | Synthesizes 2 years of daily demand (5 stores) and supplier capacity (3 suppliers), since no real dataset was available |
| Forecasting | `forecasting/forecast.py` | Trains **Prophet** and **ARIMA** per store, compares accuracy (MAPE/RMSE), and produces a 30-day baseline demand plan |
| Simulation | `simulation/simulate_disruption.py` | Day-by-day inventory simulation of the network under 3 scenarios: **baseline**, **supplier failure**, **port closure** |
| KPIs | `kpi/kpi_calculator.py` | Computes **service level**, **recovery time**, and **cost impact** for each disruption scenario vs. baseline |
| Dashboard | `dashboard/control_tower_dashboard.html` | Interactive control-tower view — open this file directly in any browser |

All intermediate and final data is in `outputs/*.csv` so you can inspect every
step without re-running anything.

## 2. How to run it yourself

```bash
pip install prophet statsmodels pandas numpy
python3 data/generate_data.py
python3 forecasting/forecast.py
python3 simulation/simulate_disruption.py
python3 kpi/kpi_calculator.py
```

Then open `dashboard/control_tower_dashboard.html` in a browser — no server
needed, the data is embedded in the file.

## 3. The methodology, explained

### Step 1 — Demand data
Real company data wasn't available, so demand was synthesized to look like
real retail sales: a slow upward trend, a weekend spike (people shop more on
Sat/Sun), a yearly seasonal wave (a mid-year peak), and random day-to-day
noise. Each of the 5 stores got its own profile (different size and
volatility) so the network isn't perfectly uniform.

### Step 2 — Forecasting
For each store, the last 60 days of history were held out as a test set.
**Prophet** (additive trend + weekly + yearly seasonality) and **ARIMA**
(classical autoregressive model, with the (p,d,q) order chosen by a small
grid search over AIC) were both trained on the remaining history and scored
on the holdout using **MAPE** (Mean Absolute Percentage Error) and **RMSE**.

Result: **Prophet outperformed ARIMA in every store** (roughly 4-6% MAPE vs.
7-9% MAPE), because the synthetic demand has strong weekly seasonality that
Prophet models natively, while ARIMA has to infer that pattern purely from
lagged values. Prophet's forecast for each store became the **30-day
baseline demand plan** — this is the "no disruption" reference plan used in
the next step.

### Step 3 — Disruption simulation
This is the core "control tower" logic: a day-by-day loop that tracks
warehouse inventory.

- Each day, suppliers ship in their daily capacity → added to warehouse inventory
- Store demand is subtracted from inventory
- If inventory can't cover demand, that shortfall is "unfulfilled demand" (a stockout)
- If inventory falls below a safety-stock threshold, the network can
  **expedite** extra supply (e.g. air freight, spot-market buys) — but only up
  to a limited daily amount, at a cost premium — this is what makes a
  disruption **actually bite** instead of being fully absorbed

Two disruption scenarios were modeled, both lasting 14 days starting on day 20:
- **Supplier failure**: the largest supplier (38% of network capacity) drops to zero output
- **Port closure**: all three suppliers' inbound shipments are delayed, so effective capacity drops sharply and then arrives in a catch-up surge once the port reopens

### Step 4 — Resilience KPIs
For each scenario, three KPIs are computed against the baseline run:

1. **Service Level** — % of demand fulfilled. Reported as both the overall
   average and the "trough" (worst single day) — the trough is the number
   an ops team actually worries about.
2. **Recovery Time** — days from the start of the disruption until service
   level returns to and stays at ≥99%. This is the classic resilience-curve
   metric (how deep the dip, how long the recovery).
3. **Cost Impact** — extra $ spent vs. baseline, split between expediting
   premiums and lost-sale costs (unfulfilled demand assumed to cost 2x unit
   cost in lost margin + goodwill).

A **Network Robustness Score** (0-100) was also added as a composite index —
this is **not** a standard industry metric, it's a simple weighted blend of
the three KPIs (40% service level, 30% recovery time, 30% cost) so scenarios
can be ranked at a glance. Be upfront in interviews that this is a metric you
designed, not an industry standard.

### Results (from the synthetic data)

| Scenario | Trough Service Level | Recovery Time | Extra Cost | Robustness Score |
|---|---|---|---|---|
| Supplier Failure | 72.9% | 14 days | +$15,939 (3.4%) | 100/100 |
| Port Closure | 57.5% | 14 days | +$24,720 (5.3%) | 30/100 |

**Takeaway**: the port closure is the more damaging scenario in this network
— it hits every supplier at once rather than just one, so the warehouse has
no unaffected supply route to lean on, causing a deeper stockout and more
expensive emergency expediting.

## 4. Assumptions made (be ready to state these explicitly)

- Starting warehouse inventory = 3 days of average network demand (a "lean"
  network, chosen deliberately so a disruption produces a visible effect —
  a network with 30 days of buffer stock wouldn't show anything interesting)
- Safety stock threshold = 2 days of average demand
- Emergency/expedited capacity is capped at 20% of average daily demand per
  day (representing limited air-freight/spot-market capacity)
- Expediting costs 40% more per unit than normal; a lost sale costs 2x unit
  cost (assumed margin + goodwill)
- All numbers (costs, capacities, lead times) are assumed/synthesized —
  clearly label this as illustrative when presenting it, not real company data

## 5. How to talk about this in an interview

Be honest that the data is synthetic (interviewers respect this far more
than pretending it's real) but walk through the **method** as if it were
real: "I built a simulation that models a lean 2-echelon network, and used
it to quantify — in dollars and days, not just qualitatively — how much
worse a port closure is than a single supplier going down. That's the kind
of business case a resilience/procurement team would use to justify
diversifying suppliers or holding more buffer stock at a specific node."
