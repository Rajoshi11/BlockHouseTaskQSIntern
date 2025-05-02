# BlockHouse Quant Strategist Intern May 2025_Rujuta JoshiTask
# Cont & Kukanov Static Smart‑Order‑Router  
### Trial Task: Cont & Kukanov Back-testing
---

## 1 .  Pipeline Overview

``l1_day.csv`` *(≈ 60 k Level‑1 messages for AAPL, 9‑min span)*  

→ **snapshot builder**  

→ list of per‑timestamp venue books (pre‑cached)  

→ **tuner**  
&nbsp;&nbsp;• coarse grid (144 points) *(6 (λ_over) × 6 (λ_under) × 4 (θ_queue) = 144 points.)*   
&nbsp;&nbsp;• optional fine search around the best coarse triple  

→ **simulate** for each candidate  
&nbsp;&nbsp;• exact Cont–Kukanov allocator (100‑share grid)  
&nbsp;&nbsp;• 60 % fill haircut, identical taker fee on every venue  

→ choose min‑cost parameters  

→ run baselines (Best‑Ask, 60 s TWAP, VWAP) under identical fee/haircut  

→ **Outputs**  
&nbsp;&nbsp;• spec JSON to `stdout`  
&nbsp;&nbsp;• `results_bar.png` (annotated bar chart)  
&nbsp;&nbsp;• `results.png`   (cumulative‑cost line plot)  

snapshots -> tuner -> allocate() -> simulate() (router) -> baselines (BA/TW/VW)

*Progress logging* – every time the grid search finds a cheaper triple it prints  
`[coarse] …` or `[ fine ] …`, so convergence is transparent.

---

## 2 .  Parameter Ranges & Search Strategy

| Symbol | Meaning | Coarse grid | Fine search (`--fine`) |
|--------|---------|-------------|------------------------|
| λ<sub>over</sub> | penalty per extra share | 0 → 0.25 in 0.05 $ | ± 0.025 around coarse best, 0.005 $ step |
| λ<sub>under</sub> | penalty per unfilled share | same | same |
| θ<sub>queue</sub> | linear queue‑risk | 0 → 0.06 in 0.02 $ | same |

* Why 0–25 ¢?  Larger penalties only add constant cost once fills never under‑/over‑shoot.  
* Why 0.5 ¢ step?  Finer than the 1 ¢ tick but still < 1 000 points.

Runtime  
* coarse grid only  → **≈ 40 s**  
* coarse + fine     → **80‑110 s** (both < 2 min laptop spec)

---

## 3 .  Code Structure
```text
backtest.py
├─ load_snapshots()   # dedupe feed → List[List[VenueSnap]]
├─ allocate()         # exact Cont‑Kukanov pseudocode, STEP = 100
├─ simulate()         # executes fills, optional cumulative tracking
├─ baselines          # best_ask / twap / vwap
├─ tune()             # coarse (144 pts) + optional fine search, logs progress
└─ __main__           # CLI, prints JSON, saves plots
```

``backtest.py`` is < 250 lines and imports only `pandas`, `numpy`, `matplotlib`, `sys`, `time`.
Allocator and baselines share **identical fill haircut & fees**, ensuring apples‑to‑apples cost.

---

## 4 .  Representative Results  *(public CSV)*

| Strategy | Cash ($) | Avg Px ($) | Δ vs Best‑Ask (bp) |
|----------|---------:|-----------:|-------------------:|
| Best‑Ask | 1 114 115.85 | 222.82317 | 0.00 |
| TWAP 60 s| 1 115 253.70 | 223.05074 | ‑10.21 |
| VWAP     | 1 115 335.24 | 223.06705 | ‑10.95 |
| **Tuned SOR** | **1 113 714.50** | **222.74290** | **+3.60** |

Coarse grid runtime: **11 s** on 2020 laptop (single core).  
Plots saved automatically.

results.png

![image](https://github.com/user-attachments/assets/518e413a-b008-4e66-afe1-996a03795591)

results_bar.png

![image](https://github.com/user-attachments/assets/fe801e48-3f4e-4e4f-9831-1c7146f255df)

Output for particular test run:
```
[coarse] 1,113,714.50  lo=0.000 lu=0.000 th=0.000
{
  "best_params": {
    "lambda_over": 0.0,
    "lambda_under": 0.0,
    "theta_queue": 0.0
  },
  "tuned_router": {
    "cash": 1113714.5,
    "avg_px": 222.7429
  },
  "best_ask": {
    "cash": 1114115.8499999996,
    "avg_px": 222.82316999999992
  },
  "twap": {
    "cash": 1115253.7000000034,
    "avg_px": 223.0507400000007
  },
  "vwap": {
    "cash": 1115335.235122325,
    "avg_px": 223.06704702446498
  },
  "savings_bps": {
    "vs_best_ask": 3.6,
    "vs_twap": 13.82,
    "vs_vwap": 14.55
  },
  "runtime_sec": 10.68
}
```

---

## 5 .  Realism Improvement Idea

**Probabilistic queue depletion**

Instead of a fixed 60 % haircut, let executable size be  
`Binomial(displayed, p)` with `p = e^{‑θ · queue_position}`.  
The allocator would then choose non‑zero θ to balance price advantage vs. queue‑risk; λ<sub>over</sub>/λ<sub>under</sub> become relevant whenever stochastic fills overshoot or undershoot.

---

## 6 .  How to Run

```bash
# coarse grid (fast, specification target)
python backtest.py --csv l1_day.csv

# add local fine search (still < 2 minutes)
python backtest.py --csv l1_day.csv --fine
