# Product Requirements Document
## P2P Local Energy Trading Platform

**Version:** 1.0.0  
**Last Updated:** 2025-06-02  
**Context:** South Korean microgrid market — currency ₩ KRW, timezone KST (UTC+9)  
**Deployment:** cPanel Shared Hosting — Python 3.12.13, Flask + Passenger WSGI  
**Grid Operator:** Grida Smartgrid  
**Institution:** Energy ICT Laboratory, Chonnam National University

---

## 1. Product Overview

A Peer-to-Peer (P2P) local energy trading platform acting as a **Microgrid Market Matchmaking Engine**. Prosumer households equipped with solar panels and battery storage trade surplus energy with neighboring buyers using a Double-Auction algorithm. AI-driven bid generation via ARIMA forecasting is implemented and operational.

### Phase Status

| Phase | Name | Status |
|---|---|---|
| Phase 1 | Data Ingestion & Visualization | ✅ Complete |
| Phase 2 | Rule-Based Matching Engine | ✅ Complete |
| Phase 3 | AI-Driven Bid Automation | ✅ Complete |

---

## 2. Technical Stack

Original blueprint specified FastAPI + Uvicorn. Revised to Flask due to cPanel Passenger WSGI ASGI incompatibility.

| Layer | Original Blueprint | Current Implementation |
|---|---|---|
| Backend framework | FastAPI + Uvicorn | **Flask + Passenger WSGI** |
| Frontend | React / Vue | **Vanilla HTML/CSS/JS** |
| Charts | Recharts | **Chart.js 4.4.1 (CDN)** |
| Data engine | Pandas | **Pandas** |
| AI engine | Chronos / LightGBM / DRL | **ARIMA — statsmodels** |
| Hosting | Any server | **cPanel Shared Hosting** |
| CORS | — | **flask-cors** |

### Installed Packages
```
flask
flask-cors
pandas
statsmodels
```

---

## 3. Data Schema

### A. Session Sell Orders (`session_sell.csv`)
User-submitted sell orders per hour. Supports **multi-tier bidding** — multiple rows for the same user and timeslot at different prices.

| Column | Type | Description |
|---|---|---|
| `user` | String | Unique household identifier |
| `Time(KST)` | DateTime | Hourly timestamp (YYYY-MM-DD HH:MM) |
| `Power_sell(Wh)` | Integer | Energy volume offered for sale |
| `Price(Won)` | Integer | Minimum acceptable price per Wh |
| `Tolerance(%)` | Integer | % seller willing to lower price to clear inventory |

### B. Session Buy Orders (`session_buy.csv`)
User-submitted buy orders per hour. Also supports multi-tier bidding.

| Column | Type | Description |
|---|---|---|
| `user` | String | Unique household identifier |
| `Time(KST)` | DateTime | Hourly timestamp (YYYY-MM-DD HH:MM) |
| `Power_buy(Wh)` | Integer | Energy volume requested for purchase |
| `Price(Won)` | Integer | Maximum acceptable price per Wh |
| `Tolerance(%)` | Integer | % buyer willing to overpay during scarcity |

### C. Prosumer History (`history_userX_prosumer.csv`)
7-day hourly history used for ARIMA training and dashboard visualization.

| Column | Type | Description |
|---|---|---|
| `user` | String | Unique household identifier |
| `Time(KST)` | DateTime | Hourly timestamp (YYYY-MM-DD HH:MM) |
| `Solar_gen(Wh)` | Integer | Solar panel output per hour |
| `Consumption(Wh)` | Integer | Household energy consumption per hour |
| `Battery_level(Wh)` | Integer | Current battery charge level |
| `Battery_max(Wh)` | Integer | Maximum battery capacity |
| `Power_sell(Wh)` | Integer | Energy sold (0 if not selling) |
| `Power_buy(Wh)` | Integer | Energy bought (0 if not buying) |
| `Price(Won)` | Integer | Bid price per Wh |
| `Tolerance(%)` | Integer | Acceptable price deviation |
| `Action` | String | `sell` / `buy` / `idle` |

### Battery Decision Logic
```
Solar surplus?
  YES → Charge battery first
        Battery ≥ 80% full? → SELL excess to peers
        Battery < 80%       → IDLE (keep charging)
  NO  → Discharge battery to cover deficit
        Battery > minimum?  → IDLE (self-sufficient)
        Battery at minimum  → BUY from peers
```

---

## 4. Matching Engine — Double-Auction Algorithm

File: `matching_engine.py`

```
For each hourly timeslot:
  1. Compute price bounds:
       Seller min = Price × (1 - Tolerance / 100)
       Buyer  max = Price × (1 + Tolerance / 100)
  2. Sort sellers ascending by min price
     Sort buyers  descending by max price
  3. Match while Buyer max ≥ Seller min:
       Matched volume  = min(Power_sell, Power_buy)
       Clearing price  = midpoint(Seller min, Buyer max)
  4. Unmatched residuals → Grida fallback:
       Surplus → export at SMP rate  (₩/Wh, operator-configurable)
       Deficit → import at retail rate (₩/Wh, operator-configurable)
```

### Multi-Tier Bidding
Multiple rows for the same user and timeslot are supported natively. The engine processes each row as an independent order and matches cheapest sell tiers first.

### Matching Failure Conditions
| Cause | Description |
|---|---|
| Timeslot mismatch | Sell and buy orders have no hours in common |
| Price gap | Seller min price > buyer max price at overlapping hours |
| Zero tolerance | Tolerance = 0% leaves no room for price negotiation |

### Financial Metrics per User (Leaderboard)

| Metric | Formula |
|---|---|
| `p2p_saving` | `(clearing−SMP)×sell_wh + (retail−clearing)×buy_wh` |
| `gross_revenue` | `sell_p2p_revenue + sell_grida_revenue` |
| `total_buy_cost` | `buy_p2p_cost + buy_grida_cost` |
| `net_position` | `gross_revenue − total_buy_cost` |
| `grida_profit` | `grida_import_value − grida_export_value` |

### Composite Score (0–100)
```
score = 0.40 × match_rate + 0.30 × saving_efficiency + 0.30 × volume_share
```

---

## 5. API Endpoints

All endpoints live under the Flask app root. CORS enabled for all `/api/*` routes.

| Method | Endpoint | Description |
|---|---|---|
| GET | `/` | Serves dashboard HTML |
| GET | `/api/status` | Health check |
| POST | `/api/upload-csv` | Upload prosumer history CSV → returns stats + hourly data |
| POST | `/api/match` | Upload session_sell + session_buy (multipart) → run matching |
| POST | `/api/match-json` | Same as above but accepts JSON `{sell_csv, buy_csv}` — used by AI matching from browser memory |
| POST | `/api/train` | Upload history CSV as JSON `{csv_string}` → fit ARIMA → save `.pkl` |
| GET | `/api/predict/<user>` | Load trained model → return 24h forecast + session rows |
| GET | `/api/predict/<user>/download` | Download AI session CSVs as ZIP |

### Grida Rate Configuration
SMP and retail rates are passed as form parameters to `/api/match` and `/api/match-json`:
```
smp_rate    (default: 900  ₩/Wh) — price Grida pays sellers for unmatched surplus
retail_rate (default: 1300 ₩/Wh) — price Grida charges buyers for unmatched deficit
```

---

## 6. Frontend Dashboard

**File:** `templates/index.html`  
**Stack:** Vanilla HTML/CSS/JS + Chart.js 4.4.1  
**API config:** Auto-detects `file://` vs `http://` — set `SERVER_URL` once for remote access

### Tab Structure

| Tab | Name | Function |
|---|---|---|
| 1 | ⚡ Matching Engine | Upload session CSVs, set Grida rates, run Double-Auction, view leaderboard + Grida card |
| 2 | 📂 History & Dataset | Upload prosumer history, visualize 4 charts per day, hourly table |
| 3 | 🤖 AI Forecast | Auto-uses Tab 2 data → Train ARIMA → Preview 24h forecast → Download session CSV |
| 4 | 📈 Comparison | Run AI matching (replaces trained user's rows), side-by-side Manual vs AI results |

### Tab 1 — Matching Engine Features
- Quick guide: 4 info cards (how matching works, failure causes, multi-tier, Grida rates)
- Dual upload zones (sell + buy CSV)
- Grida rate inputs (SMP + retail, editable)
- Zero-trade warning with auto-diagnosis (timeslot mismatch vs price gap)
- Summary metrics: trades, matched Wh, avg clearing, Grida export/import
- Grida net position card (revenue, cost, net gain/loss)
- Matched trades table with pagination
- Grida export + import residual tables
- User leaderboard: score bar, match rate, P2P saving, net position, gross revenue, buy cost

### Tab 3 — AI Forecast Features
- No file upload needed — uses history from Tab 2 automatically
- ARIMA(2,1,2) trained on `Price(Won)` column
- Hourly average sell/buy volumes from history used for volume prediction
- Tolerance auto-set based on price volatility (std dev)
- Bar chart: predicted prices colored by action (green=sell, orange=buy)
- Download AI session CSVs as ZIP

### Tab 4 — Comparison Features
- Run AI Matching button — replaces trained user's rows in Tab 1 session
- Focused user card: Manual vs AI head-to-head (6 metrics, winner per metric)
- Overall session comparison table with Grida net
- Grida side-by-side: Manual session vs AI session (revenue, cost, net)
- Full session leaderboard with sort buttons (Manual Score / AI Score / Δ Score)
- General insight note explaining how to read results
- Columns: Manual/AI score, match%, P2P saving, net position, gross revenue, buy cost

---

## 7. ARIMA Forecasting

**Model:** `ARIMA(2, 1, 2)` via `statsmodels`  
**Target variable:** `Price(Won)` hourly time series  
**Training data:** 7-day history (168 observations minimum)

### Training Strategy
```
Dataset < 5 MB  → train on server via POST /api/train
Dataset ≥ 5 MB  → train locally → upload .pkl to models/
```

### Prediction Logic
```python
# Action per hour:
if price > avg_price × 1.03 and history shows selling this hour → SELL
if price < avg_price × 0.97 and history shows buying  this hour → BUY
else → IDLE

# Volume: historical average Wh per hour-of-day
# Tolerance: 15% if price std dev > ₩80, else 10%
```

### Model Files
One `.pkl` file per user: `models/arima_user1.pkl`, `models/arima_user2.pkl`, etc.  
Each `.pkl` contains: fitted model, avg_price, std_price, hourly_avg_wh, metadata.

---

## 8. Sample Datasets

| File | Description | Users |
|---|---|---|
| `session_sell_v3.csv` | Recommended — user1 intentionally suboptimal manually | 5 |
| `session_buy_v3.csv` | Counterpart to v3 sell, good P2P overlap | 5 |
| `session_sell_multitier.csv` | Multi-tier bidding (3 price tiers per hour) | 5 |
| `session_buy_multitier.csv` | Multi-tier buy orders with emergency tier | 5 |
| `session_sell_nomatch.csv` | No-match demo: price gap + timeslot mismatch | 3 |
| `session_buy_nomatch.csv` | No-match counterpart | 2 |
| `history_user1_v3.csv` | Recommended — clean solar pattern for ARIMA | 1 |
| `history_user1_prosumer.csv` | Original — scattered buy hours | 1 |

---

## 9. Known Constraints & Design Decisions

| Item | Detail |
|---|---|
| No persistent process | Passenger WSGI restarts on idle — no background schedulers |
| No GPU | ARIMA inference only; no deep learning on server |
| ARIMA not DRL | Phase 3 DRL (Stable-Baselines3) not feasible on shared hosting |
| Single-user AI | AI replaces one user at a time; multi-user AI requires training multiple models |
| Clearing price = midpoint | Fair split between seller min and buyer max; weighted midpoint is a future option |
| CORS origins = `*` | Open for development; restrict to specific domains for production |
| `TEMPLATES_AUTO_RELOAD = True` | HTML changes reflect on browser refresh without restart |
| File protocol CORS | `file://` opens use `SERVER_URL` config; `http/https` use same origin |
| Multipart unreliable on Passenger | `/api/train` and `/api/match-json` use JSON body instead of multipart |

---

## 10. File Structure

```
p2p_energy_api/                    ← cPanel app root
├── main.py                        ✅ Flask app + all API endpoints
├── matching_engine.py             ✅ Double-Auction + leaderboard + profit
├── train.py                       ✅ ARIMA training script
├── predict.py                     ✅ ARIMA inference + session CSV generation
├── requirements.txt               ✅ flask, flask-cors, pandas, statsmodels
├── PRD.md                         ✅ This document
├── templates/
│   └── index.html                 ✅ 4-tab dashboard frontend
├── models/
│   └── arima_userX.pkl            ✅ Trained ARIMA models (per user)
└── data/
    ├── session_sell_v3.csv
    ├── session_buy_v3.csv
    ├── session_sell_multitier.csv
    ├── session_buy_multitier.csv
    ├── session_sell_nomatch.csv
    ├── session_buy_nomatch.csv
    ├── history_user1_v3.csv
    └── history_user1_prosumer.csv
```
