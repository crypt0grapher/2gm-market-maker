Below is a **stand‑alone Markdown software‑development plan** that an autonomous Python agent can follow to build a **Jupyter‑notebook market‑making bot** implementing the *hybrid Grossman–Miller + Glosten–Milgrom model* with Binance (or any CCXT‑compatible CEX) order‑book data.

---

## **0  Document Map**

| §  | Section                   | Purpose                               |
| -- | ------------------------- | ------------------------------------- |
|  1 | Scope & success criteria  | Defines the “definition of done”      |
|  2 | Architecture overview     | High‑level modules & data flow        |
|  3 | Detailed module specs     | Exact APIs, classes & functions       |
|  4 | Notebook outline          | Ordered cell plan for the `.ipynb`    |
|  5 | Implementation workflow   | Task list & recommended timeline      |
|  6 | Testing & validation plan | Unit, integration & performance tests |
|  7 | Ops & extension hooks     | Keys, secrets, deployment, next steps |

---

## **1  Project Scope & Success Criteria**

### 1.1 Goals

1. **Live ingest** level‑2 order‑book + trade prints for a target symbol (default `BTC/USDT`) via **CCXT / CCXT Pro**.
2. **Estimate model parameters** in real time:

   * σ (spot volatility)
   * γ (dealer risk aversion)
   * k (information–toxicity coefficient)
   * τₜ (rolling toxicity score, VPIN)
3. **Generate quotes** each loop using

$$
\begin{aligned}
\delta_t^{\text{info}} &= k \,\tau_t \\
\psi_t &= \kappa\,Q_t,\quad \kappa=\dfrac{\gamma\sigma^2\Delta}{n} \\
a_t &= m_{t-1} + \delta_t^{\text{info}} - \psi_t  \\
b_t &= m_{t-1} - \delta_t^{\text{info}} - \psi_t
\end{aligned}
$$

4. **Paper‑trade simulation** (fill on touch) or **live post‑only orders** (`create_order` with `{"postOnly":True}` on Binance).
5. **Persist** raw data, fills, and PnL to Parquet/CSV.
6. **Display KPIs** in‑notebook: cumulative PnL, Sharpe, inventory histogram, average captured spread.

### 1.2 Out‑of‑scope

* Reinforcement learning, grid strategies, GUI.
* Cross‑venue hedging (add later).

### 1.3 Definition of Done

* Notebook runs end‑to‑end with **no manual edits**, connects, streams live data for ≥ 10 minutes, and prints live KPIs.
* All critical functions covered by unit tests inside the notebook.
* Code style: `flake8` clean, PEP‑8.

---

## **2  Architecture Overview**

```
┌──────────────┐
│Config (YAML) │ <- API keys, symbol, risk caps
└──────┬───────┘
       │
┌──────▼───────┐    CCXT / CCXT Pro
│ Exchange API │◄─────────────────────────┐
└──────┬───────┘                          │
       │        Level‑2 / trades          │
┌──────▼────────────┐                     │
│ Data Ingestion    │ --raw→  Parquet ←───┘
└──────┬────────────┘
       │ cleaned     ┌─────────┐
       │             │Feature  │  σ, τ
       │             │Builder  │
       │             └─────────┘
       │                     ▼
┌──────▼────────────┐   ┌────────┐
│Parameter Estimator│   │Inventory│ Qₜ
└──────┬────────────┘   └────────┘
       │ κ, k                │
┌──────▼────────────┐        │
│Quote Engine       │ mₜ,bₜ,aₜ
└──────┬────────────┘        │
       │ create/cancel       │
┌──────▼────────────┐        │
│Order Manager      │◄───────┘
└──────┬────────────┘
       │ fills
┌──────▼────────────┐
│Backtest & Metrics │ → charts / tables
└───────────────────┘
```

---

## **3  Module‑by‑Module Specification**

| #    | Module             | Responsibility                                                                                                                 | Key APIs                                                                               |
| ---- | ------------------ | ------------------------------------------------------------------------------------------------------------------------------ | -------------------------------------------------------------------------------------- |
| 3.1  | **`settings.py`**  | Load YAML/ENV for keys, symbol, risk caps, file paths.                                                                         | `load_config(path:str)->dict`                                                          |
| 3.2  | **`connect.py`**   | Instantiate `ccxt.binance()` (*REST*) and `ccxt.pro.binance()` (*WS*). Enable `rateLimit=True`.                                | `get_exchange(async_ws:bool=False)`                                                    |
| 3.3  | **`ingest.py`**    | (a) Historical back‑fill via `fetch_trades` & `fetch_order_book` (REST). (b) Live stream via `watch_order_book` ([GitHub][1]). | `async stream_orderbook(exchange,symbol,depth=20)` yields (`timestamp`,`bids`,`asks`). |
| 3.4  | **`features.py`**  | • Realised vol σ (Parkinson or std of mid‑returns).  • VPIN toxicity (bucket size = `N_trades`).                               | `update_sigma(df)->float`, `update_vpin(trades)->tau`                                  |
| 3.5  | **`params.py`**    | Online EWMA estimators for γ (from historical PnL variance) & κ.  `n` default = 10.                                            | `update_k(slip_spread, tau)->k`                                                        |
| 3.6  | **`inventory.py`** | Track signed position Qₜ, hard‑cap ±`Q_max`, auto‑hedge stub.                                                                  | `update(q_delta)`, `should_hedge()->bool`                                              |
| 3.7  | **`quotes.py`**    | Implement eq. (3); round to tick; enforce min‑notional filters ([GitHub][2]).                                                  | `make_quotes(mid, k, tau, kappa, Q)->(bid, ask)`                                       |
| 3.8  | **`orders.py`**    | Place/cancel through `create_order` (`"limit"` , `"postOnly": True`) ([GitHub][3]); ignore partial fills < `min_qty`.          | `sync_quotes(bid, ask)`                                                                |
| 3.9  | **`backtest.py`**  | Fill‑on‑touch sim for historical data; latency parameter.                                                                      | `run_backtest(data)->stats`                                                            |
| 3.10 | **`metrics.py`**   | PnL, Sharpe, mean spread capture, inventory CVaR, fill ratio.                                                                  | `calc_metrics(fills, inventory)`                                                       |
| 3.11 | **`plotting.py`**  | Matplotlib charts (one plot per figure – no seaborn).                                                                          | `plot_pnl(df)` etc.                                                                    |
| 3.12 | **`tests.py`**     | `assert`‑style tests inside nb: e.g. `assert make_quotes(...)[0] < mid`.                                                       | `run_unit_tests()`                                                                     |

---

## **4  Notebook Cell‑By‑Cell Outline**

| Cell # | Title / Tag              | Content                                                                                                                                                              |
| ------ | ------------------------ | -------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
|  0     | **preamble**             | `pip install ccxt ccxt.pro pandas pyarrow matplotlib`                                                                                                                |
|  1     | **imports**              | `import asyncio, ccxt, ccxt.pro as ccxtpro, pandas as pd, numpy as np, ...`                                                                                          |
|  2     | **config loader**        | YAML/ENV reading; print loaded params (masked key).                                                                                                                  |
|  3     | **exchange connect**     | Instantiate REST + WS objects; load markets once (`load_markets()`).                                                                                                 |
|  4     | **helper classes**       | Paste Modules 3.3–3.8 in compact form (inner classes).                                                                                                               |
|  5     | **historical back‑fill** | `await exchange.fetch_trades(symbol, since, limit)`; build mid‑price series; compute initial σ, k.                                                                   |
|  6     | **live async loop**      | Coroutine:  $a$ `ob = await ws.watch_order_book(symbol, 20)`  $b$ recompute τ, κ  $c$ make & sync quotes  $d$ update inventory/PnL  $e$ every N loops write Parquet. |
|  7     | **metrics snapshot**     | Every 1 min display PnL, Sharpe; use IPython display clear‑output.                                                                                                   |
|  8     | **shutdown handler**     | Graceful `await exchange.close()`; dump final CSV.                                                                                                                   |
|  9     | **plots**                | `plot_pnl`, inventory histogram, rolling Sharpe.                                                                                                                     |
|  10    | **unit tests**           | Call `run_unit_tests()` — all asserts must pass.                                                                                                                     |
|  11    | **backtest demo**        | Run `run_backtest` for prior 24 h, print metrics table.                                                                                                              |
|  12    | **FAQ / next steps**     | TODO list: add futures hedge, live OMS wrapper, dockerisation.                                                                                                       |

---

## **5  Implementation Workflow**

| Day | Task                                                                                                                 | Owner (agent) |
| --- | -------------------------------------------------------------------------------------------------------------------- | ------------- |
|  1  | Set up virtual env & notebook skeleton (Cells 0‑3).                                                                  | Dev agent     |
|  1  | Implement `settings.py`, `connect.py`; verify `fetch_order_book` returns dict with `bids/asks` fields ([GitHub][3]). |               |
|  2  | Build `ingest.py`; stream order book for 5 min; write raw snapshot Parquet.                                          |               |
|  2  | Implement `features.py` (σ, VPIN); cross‑check σ vs daily realised vol.                                              |               |
|  3  | Code `quotes.py` & `inventory.py`; dry‑run quote computation only (no orders).                                       |               |
|  3  | Add `backtest.py`; replay one day of data; target Sharpe > 1.0.                                                      |               |
|  4  | Integrate `orders.py`; place **post‑only** live orders on testnet; verify they rest in book.                         |               |
|  4  | Populate `metrics.py`, `plotting.py`; live dashboard in nb.                                                          |               |
|  5  | Harden error handling: WS disconnect, REST `DDoSProtection`, min‑notional failures.                                  |               |
|  5  | Finish `tests.py`; cells 10 & 11 must run green.                                                                     |               |
|  6  | Documentation pass; inline docstrings, README; push to Git.                                                          |               |
|  7  | Burn‑in 24 h unattended paper‑trade; evaluate KPIs; deliver.                                                         |               |

---

## **6  Testing & Validation**

### 6.1 Unit tests

* **Quote monotonicity** — ask ≥ bid ≥ 0.
* **Inventory shift sign** — if Qₜ > 0 then ψₜ > 0 (ask lower than bid shift).
* **VPIN range** — τₜ ∈ \[0,1].

### 6.2 Integration

* WS reconnect within 5 s on `NetworkError`.
* REST `fetch_order_book` latency < 300 ms.

### 6.3 Performance acceptance

* Paper PnL Sharpe ≥ 1.5 on BTC/USDT over 3 days.
* 95 % inventory |Q| < `Q_max`.
* Average realised spread capture ≥ 25 % of top‑of‑book spread.

---

## **7  Operational Notes & Extension Hooks**

| Topic               | Recommendation                                                                                             |
| ------------------- | ---------------------------------------------------------------------------------------------------------- |
| **API keys**        | Store in `.env`; load with `python‑dotenv`; never commit.                                                  |
| **Rate limits**     | Use `exchange.enableRateLimit = True`; honour `exchange.rateLimit`.                                        |
| **Timezone**        | Convert all timestamps to UTC using `pd.to_datetime(..., unit='ms', utc=True)`.                            |
| **Data retention**  | Raw WS snapshots → `/data/orderbook/YYYY‑MM‑DD/{hh}.parquet`.                                              |
| **Docker**          | Base image `python:3.12‑slim`, install ccxt‑pro wheel.                                                     |
| **Prod deployment** | Run notebook headlessly via `papermill` or convert to `.py` with `jupyter nbconvert --to script`.          |
| **Future work**     | Perpetual futures hedge, multi‑symbol quoting, latency optimisation (Cython), Prometheus metrics endpoint. |

---

### **Key CCXT/CCXT Pro references**

* **`fetch_order_book()`** signature & return structure – Manual §“Order Book” ([GitHub][3])
* **`create_order`** parameters (limit / `postOnly`) – Manual §“Orders” ([GitHub][3])
* **`watch_order_book`** WebSocket stream – CCXT Pro manual list of watch‑methods ([GitHub][1])

> This blueprint is intentionally exhaustive; a competent Python agent (or human) can follow it cell by cell to deliver a working hybrid‑model market‑making notebook with no further clarification required.

[1]: https://github.com/ccxt/ccxt/wiki/ccxt.pro.manual "ccxt.pro.manual · ccxt/ccxt Wiki · GitHub"
[2]: https://github.com/ccxt/ccxt/issues/4261?utm_source=chatgpt.com "Binance can't place market order. Min Notation? #4261 - GitHub"
[3]: https://github.com/ccxt/ccxt/wiki/manual?utm_source=chatgpt.com "Manual · ccxt/ccxt Wiki - GitHub"
