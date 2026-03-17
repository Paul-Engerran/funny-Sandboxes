# Endogenous Fat Tails

Liquidation cascades produce fat-tailed returns from purely Gaussian shocks. This simulator lets you see it happen — adjust leverage, correlation, impact parameters, and watch excess kurtosis and negative skewness emerge from the mechanism itself, not from the input distribution.

**[Live demo →](https://paul-engerran.github.io/funny-Sandboxes/)**

---

## Why this exists

The standard story for fat tails in finance is "the shocks themselves are heavy-tailed" — use a Student-t, a stable distribution, whatever. That's fine descriptively, but it doesn't explain *why*. The leverage-cascade mechanism (Thurner, Farmer & Geanakoplos 2012; Brunnermeier & Pedersen 2009) gives a structural answer: agents borrow, their liquidation thresholds cluster, and when price drops enough to trigger a few, the forced selling pushes price into more thresholds. The cascade is self-amplifying and self-limiting — it stops when the leveraged positions are purged.

I wanted to see this in action, not just read about it. So I built a simulator where you can crank the parameters and watch the kurtosis respond.

---

## The model

N leveraged positions, each with a size Q_i and a liquidation price P_liq_i. Daily log-return:

```
r_t = μ(L_t) + σ₀ · ε_t + cascade_impact_t       ε_t ~ N(0,1)
```

**Drift.** Depends on aggregate leverage — higher leverage → slightly higher drift (the "calm accumulation" phase):

```
μ(L_t) = μ₀ + β · L̄_t
L̄_t = Σ (L_eff_i - 1) · Q_i  /  Σ Q_i
L_eff_i = P / (P - P_liq_i)
```

**Cascade.** After the Gaussian shock, if price P' = P·exp(r) falls below some P_liq_i, those positions are forcibly liquidated. The volume hits the market:

```
P' ← P' · exp(-λ · V_liq / V_daily)
```

This may cross more thresholds → more liquidations → iterate until stable (max 80 rounds). There's an optional convex regime (`θ`, `λ₂`, `γ`) for order book depletion — off by default because with typical parameters the linear model is sufficient and easier to reason about.

**Market depth** scales with recent realized volatility: `V_daily = V_base · (1 + 2 · realized_vol / σ₀)`. This is a crude but effective way to capture that vol begets liquidity — during volatile periods, more volume is available to absorb liquidations.

**LTV correlation.** Positions can share a common factor in their liquidation thresholds:

```
z_i = ρ · z_common + √(1-ρ²) · z_idio   →   LTV_i via Φ(z_i)
```

ρ = 0 means independent thresholds. ρ = 0.6 (the default) creates "liquidation walls" — many positions with similar P_liq, which makes cascades rarer but much more violent when they happen. This felt more realistic than independent draws.

**Renewal.** Liquidated positions are replaced at the current price. Without this the model dies — the position pool shrinks and cascades stop. The renewal captures new entrants buying the dip.

**Baseline.** Every simulation also runs a Gaussian path with the same ε_t, constant μ₀, no cascade. Same randomness, different dynamics.

---

## Parameters

### Market

| Symbol | Slider | Default | Description |
|--------|--------|---------|-------------|
| N | Positions | 400 | Number of leveraged positions |
| T | Days | 500 | Simulation length |
| σ₀ | Daily vol | 0.030 | Fundamental volatility |
| μ₀ | Drift | 0.0001 | Base daily drift (~2.5% ann.) |

### Leverage

| Symbol | Slider | Default | Description |
|--------|--------|---------|-------------|
| LTV_min | LTV min | 0.55 | Lower bound of loan-to-value |
| LTV_max | LTV max | 0.83 | Upper bound — higher means closer to liquidation at entry |
| β | Drift sensitivity | 0.0002 | Leverage → drift coupling |

### Cascade

| Symbol | Slider | Default | Description |
|--------|--------|---------|-------------|
| λ | Impact | 1.4 | Price impact per unit of liquidation volume |
| depth | Market depth | 2.0 | Multiplier for daily absorbed volume |
| spread | Entry spread | 25% | Dispersion of entry prices around P₀ |
| θ, λ₂, γ | Convex params | off | Optional order book depletion — hidden behind a toggle |

### Correlation

| Symbol | Slider | Default | Description |
|--------|--------|---------|-------------|
| ρ | LTV correlation | 0.60 | Common factor loading on liquidation thresholds |

---

## What it shows

### Monte Carlo (index.html)

The main view. Runs many simulations and aggregates.

Fan chart (log scale), aggregated return distribution, QQ-plot, risk metrics (VaR, ES, tail exceedances vs Gaussian theoretical values), kurtosis/skewness distributions across sims.

The more interesting outputs:

- **Mechanism decomposition** — runs 3 regimes (Gaussian / cascade only / full model) and shows how much kurtosis and skewness each mechanism adds. Uses per-sim medians because pooled kurtosis is wildly sensitive to outlier simulations (one crash in 80k returns can push kurtosis from 2 to 300).
- **Dose-response** — Δkurtosis and Δskewness as λ increases from 0 to full value. The relationship is nonlinear — most of the kurtosis comes from the last 25% of λ.
- **MC convergence** — live plot of running kurtosis estimate during computation. Useful for checking that 100 sims is enough (it usually is for the median, not always for the mean).

### Single path (path.html)

Inspect one trajectory. Price path with cascade event markers, liquidations per day, aggregate leverage dynamics, and an event log sorted by severity ("Day 237 — 42 liquidated, 3 rounds, return -7.85%"). This is where you actually *see* a cascade happen.

---

## Choices I'd defend

**Gaussian shocks only.** The whole point. If I used Student-t inputs, any fat tails in the output could be attributed to the inputs. Gaussian shocks are the cleanest test of the cascade mechanism.

**Linear impact by default.** I implemented convex impact (order book depletion) but keep it off by default. With typical parameters, V_liq/V_daily almost never exceeds the threshold θ, so the convex regime rarely activates. The linear model already produces excess kurtosis of 1–5 depending on parameters, which is in the right ballpark for equity returns.

**Seeded PRNG.** Same seed = same result. The baseline uses the same shock sequence. This is non-negotiable for a simulator where you want to compare dynamics, not randomness.

**No GARCH.** The model already generates vol clustering endogenously — a cascade bumps realized vol → market depth adjusts → next shock has more impact. Adding exogenous GARCH would muddy the attribution. It's on the future work list as a control experiment.

---

## Limitations

Not calibrated to any real asset — this demonstrates a mechanism, not a trading strategy. Single asset, no contagion. Agents are homogeneous. The cascade is instantaneous (one day), while real cascades unfold over hours or days. The price impact model is deliberately stylized.

---

## Future work

Calibrate to empirical kurtosis/skewness of S&P 500 and BTC. Add GARCH as a control to quantify how much kurtosis the cascade adds beyond vol clustering. Multi-asset with cross-liquidation.

---

## References

- Thurner, Farmer, Geanakoplos (2012). *Leverage causes fat tails and clustered volatility*. Quantitative Finance 12(5).
- Cont, Wagalath (2013). *Fire sales forensics: Measuring endogenous risk*. Mathematical Finance 26(4).
- Brunnermeier, Pedersen (2009). *Market liquidity and funding liquidity*. Review of Financial Studies 22(6).
- Adrian, Shin (2010). *Liquidity and leverage*. Journal of Financial Intermediation 19(3).
- Bouchaud, Farmer, Lillo (2009). *How markets slowly digest changes in supply and demand*. Handbook of Financial Markets.
- Mandelbrot (1963). *The variation of certain speculative prices*. Journal of Business 36(4).

---

## Technical

Vanilla JS, Chart.js, no build step. Two self-contained HTML files. Runs client-side, no server.

```
leverage-fat-tails/
├── index.html     Monte Carlo
├── path.html      Single path
└── README.md
```
