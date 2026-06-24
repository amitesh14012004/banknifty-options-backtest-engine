
# BankNifty Short Strangle Backtest Engine

**Author:** Amitesh Srivastava — [LinkedIn](https://www.linkedin.com/in/amitesh-srivastava) · [GitHub](https://github.com/amitesh14012004)  
**Tools:** Python · Pandas · NumPy · openpyxl · Matplotlib  
**Dataset:** 10M+ rows · 1-minute BankNifty options tick data · Full year 2023  
**Runtime:** < 60 seconds end-to-end on Google Colab  

---

## What This Project Does

A fully vectorised quantitative backtesting engine for a **short strangle options strategy** on the BankNifty index. The system ingests raw 1-minute options data, selects strikes, generates entry/exit signals, monitors stop-losses in real time, computes full performance analytics, and outputs a professional Excel report — all in a single pipeline run.

This project replicates the core workflow of a quantitative research desk:
- Raw market data ingestion and cleaning
- Signal generation with defined entry/exit rules
- Risk management via hard stop-loss
- Performance attribution and reporting

---

## Strategy Logic

### What is a Short Strangle?
Simultaneously **sell** one Call (CE) and one Put (PE) option. You collect premium upfront. You profit if the underlying (BankNifty) stays between the two strike prices by expiry. Risk: sharp directional moves trigger the stop-loss.

```
Short Strangle P&L = (Entry Price − Exit Price) × Quantity
Positive when option price falls (time decay / stable market)
Negative when option price rises (directional move / volatility spike)
```

### Rules

| Parameter | Value |
|-----------|-------|
| Universe | BankNifty weekly options — Week 1 only (Mon → first Wednesday) |
| Expiry | First Wednesday of each month |
| Entry | 09:20 candle close — strike closest to Rs. 50 premium |
| Exit | 15:20 candle close OR stop-loss — whichever triggers first |
| Stop-Loss | 50% of entry price — monitored via 1-min candle **High** column |
| Lot Size | 15 units · 1 lot per leg per day · No compounding |
| Starting Capital | Rs. 10,00,000 |

### Strike Selection Logic
At exactly 09:20, scan all available CE and PE strikes. Select the one whose 1-minute close price is **closest to Rs. 50** using vectorised `groupby + idxmin()` — no loops.

---

## Results — 2023 Backtest

| Metric | Value |
|--------|-------|
| CAGR | 3.70% |
| Max Drawdown | −0.22% |
| Win Rate (Combined) | 55.77% (32W / 20L) |
| CE Win Rate | 57.69% (15W / 11L) |
| PE Win Rate | 53.85% (14W / 12L) |
| Total Trades (legs) | 52 |
| Trading Days | 26 |
| Starting NAV | 100.00 |
| Ending NAV | 100.3758 |
| Total Gross P&L | Rs. 3,757.95 |
| Code Runtime | 48.4 seconds |

### Monthly P&L

| Month | P&L % | Month | P&L % |
|-------|--------|-------|--------|
| Jan 2023 | +0.05% | Jul 2023 | −0.15% |
| Feb 2023 | −0.08% | Aug 2023 | +0.02% |
| Mar 2023 | +0.02% | Sep 2023 | +0.11% |
| Apr 2023 | +0.04% | Oct 2023 | +0.22% |
| May 2023 | +0.05% | Nov 2023 | +0.03% |
| Jun 2023 | +0.02% | Dec 2023 | +0.06% |

**10 profitable months out of 12.** Only February (double SL on expiry day) and July (3 consecutive SL triggers) were loss months.

### Why Only 26 Trading Days?
Week 1 = Monday through first Wednesday of each month. 3 days were NSE market holidays:
- **Apr 4** — Mahavir Jayanti
- **May 1** — Maharashtra Day / Labour Day  
- **Oct 2** — Gandhi Jayanti

These are correctly excluded since no options data exists for those dates.

---

## Project Architecture

```
banknifty-options-backtest-engine/
│
├── backtest.py                  # Main backtest script (run this)
├── backtest_colab.ipynb         # Google Colab notebook version
│
├── data/
│   └── BANKNIFTY_SPOT.csv       # BankNifty spot price data
│   └── Options_data_2023.csv    # Options tick data (not included — see below)
│
├── outputs/
│   └── BankNifty_ShortStrangle_Backtest.xlsx   # Generated on run
│
└── README.md
```

---

## Pipeline — Step by Step

```
Raw CSV (10M rows)
      │
      ▼
[STEP 1] Load options data          ~36s  (latin1 encoding, 626 MB)
      │
      ▼
[STEP 2] Load spot data             ~1s
      │
      ▼
[STEP 3] Identify Week-1 dates      ~0.1s  (ISO week logic + expiry calendar)
      │
      ▼
[STEP 4] Strike selection           ~6s   (vectorised groupby + idxmin)
         → For each (date, CE/PE): pick strike closest to Rs.50
      │
      ▼
[STEP 5] Exit logic + SL detection  ~2.7s  (High column SL check per leg)
         → SL if any bar High ≥ Entry × 1.50
         → Else exit at 15:20 close
      │
      ▼
[STEP 6] Build tradesheet           ~0.1s
         → Quantity, Entry/Exit Value, Gross P&L, NAV, Cumulative P&L
      │
      ▼
[STEP 7] Statistics                 ~0.01s
         → CAGR, Max Drawdown, Win Rate, Avg P&L (expiry vs non-expiry)
      │
      ▼
[STEP 8] Charts                     ~0.6s
         → Equity curve + drawdown chart (matplotlib)
      │
      ▼
[STEP 9] Excel report               ~0.1s
         → 4 sheets: Guide · Tradesheet · Statistics · Charts
      │
      ▼
TOTAL: 48.4 seconds  ✓ (target: < 60s)
```

---

## Excel Output — 4 Sheets

| Sheet | Contents |
|-------|----------|
| **Guide** | Strategy documentation, formulas, config, timing breakdown |
| **Tradesheet** | 52 rows — one per option leg. Entry/Exit Date & Time, Ticker, Strike, Type, Price, SL Hit, Quantity, Value, Gross P&L, Cumulative P&L, NAV, Spot Close |
| **Statistics** | CAGR, Max Drawdown, Winners/Losers (CE/PE/Combined), Avg P&L% (expiry vs non-expiry), Monthly P&L table, Code timing |
| **Charts** | Equity curve (trade-wise NAV=100) + Drawdown chart embedded as high-res images |

---

## Key Technical Decisions

### Why Vectorisation?
With 10M+ rows, row-by-row loops take 10–15 minutes. Vectorised pandas operations (`groupby`, `idxmin`, boolean masking) complete the same work in under 60 seconds — a **15× speedup**.

### Why Use High Column for SL?
We **sold** options first (short position). A rising option price is a loss. Using the candle `High` instead of `Close` for SL detection is conservative and realistic — it catches intraday spikes that the closing price would miss, preventing understated losses.

### Why Week 1 Only?
Weekly options in the first week of the month carry the highest time-decay (theta) relative to premium — the strategy's primary profit driver. Limiting to Week 1 also reduces overnight risk across expiry cycles.

### Why Rs. 50 Premium Target?
Near-the-money options at ~Rs. 50 are liquid (tight bid-ask spread), provide meaningful premium to collect, and give the underlying room to move without immediately hitting SL — balancing risk and reward.

---

## How to Run

### Option 1 — Google Colab (recommended)
```python
# 1. Upload Options_data_2023.csv and BANKNIFTY_SPOT.csv to Google Drive
# 2. Open backtest_colab.ipynb in Colab
# 3. Update file paths in Cell 3:
OPTIONS_PATH = "/content/drive/MyDrive/Options_data_2023.csv"
SPOT_PATH    = "/content/drive/MyDrive/BANKNIFTY_SPOT.csv"
# 4. Run all cells — Excel downloads automatically
```

### Option 2 — Local
```bash
git clone https://github.com/amitesh14012004/banknifty-options-backtest-engine
cd banknifty-options-backtest-engine
pip install pandas numpy openpyxl matplotlib
# Add your data files to /data/
python backtest.py
```

### Speed Tip — Convert to Parquet (one-time)
```python
import pandas as pd
df = pd.read_csv("Options_data_2023.csv", encoding='latin1')
df.to_parquet("Options_data_2023.parquet", index=False)
# Update OPTIONS_PATH to .parquet → reduces load time from 36s to ~4s
```

---

## Data

| File | Rows | Size | Source |
|------|------|------|--------|
| Options_data_2023.csv | 10,266,681 | 626 MB | NSE 1-min options data |
| BANKNIFTY_SPOT.csv | 136,324 | ~8 MB | NSE BankNifty spot index |

> **Note:** The options data file (626 MB) is not included in this repo due to size limits.  
> The spot data file (`BANKNIFTY_SPOT.csv`) is included.  
> Options data columns: `Date, Ticker, Time, Open, High, Low, Close, Call/Put`  
> Spot data columns: `ts, o, h, l, c`

---

## Interview Talking Points

If asked about this project in a quant / DA interview:

**"What is a short strangle?"**  
Selling an OTM call and OTM put simultaneously. You profit from theta decay when the underlying stays range-bound. Max profit = combined premium collected. Risk = unlimited if underlying moves sharply past breakeven.

**"How did you handle stop-loss?"**  
Used the `High` column of each 1-minute bar, not the Close. Since we're short, a rising option price hurts us — checking High rather than Close is more realistic and conservative, catching intraday spikes.

**"Why 26 trading days instead of more?"**  
Only Week 1 (Mon–Wed) is traded. 3 of those days were NSE market holidays (Mahavir Jayanti, Labour Day, Gandhi Jayanti) — correctly excluded because no data exists for those dates.

**"What would you change to improve the strategy?"**  
(1) Dynamic strike selection based on delta instead of fixed premium target  
(2) Adjust lot size based on VIX regime — reduce exposure in high-volatility periods  
(3) Implement trailing SL instead of hard 50% — allows profitable trades more room

---

## Skills Demonstrated

| Category | Specifics |
|----------|-----------|
| Data Engineering | Large-scale CSV ingestion (10M rows), vectorised processing, Parquet optimisation |
| Quantitative Finance | Options strategy design, strike selection, SL logic, CAGR, drawdown, NAV |
| Python | Pandas vectorisation, NumPy, Matplotlib, openpyxl |
| Reporting | 4-sheet Excel with embedded charts, colour-coded P&L, frozen panes |
| Performance | Sub-60s runtime on 626 MB dataset without any distributed computing |
