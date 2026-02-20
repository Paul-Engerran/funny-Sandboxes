# Leverage & Fat Tails — Interactive Toy Lab

A small interactive “mini-lab” (single-page HTML + Chart.js) built to **visualize intuition** about how:
- **fat tails** (Gaussian vs Student-t),
- and a **stylized leverage effect** (negative shocks → stronger left tail / higher downside risk)

can impact:
- the return distribution (histograms),
- tail risk metrics (VaR / ES),
- a simple “liquidation probability” based on a threshold `S_liq`,
- and option price **distortions** (smile / term structure) relative to a Black–Scholes baseline.

## ⚠️ Scope & limitations (please read)
- This is an **educational / communication tool** (intuition builder). It is **not** an identification strategy and is **not** used to produce empirical results.
- It is **not risk-neutral** except where explicitly stated (e.g., Gaussian / Black–Scholes baseline).
  Student-t and “Student-t + leverage” are **stylized stress models** (ad hoc).
- Outputs are **stochastic** and will vary from run to run (unless a seed is implemented).
- **Not for trading** or production risk management.

## Run locally
You can open `index.html` directly in a browser.
