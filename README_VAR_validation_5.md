# Equity Option Model Validation - `VAR validation (5).ipynb`

## Project question

Can one constant-volatility Black-Scholes model, calibrated to the listed call whose strike is closest to the stock price, reproduce the remaining call-option chain? What one-day 97.5% VaR and CVaR does the same calibrated model imply for one long call contract?

This README reports the code, outputs and graph stored in the uploaded version 5 notebook. The results were not recomputed or replaced with another notebook version.

## What the notebook does

1. Downloads adjusted equity closes and call-option chains from `yfinance`.
2. Estimates annualized volatility from the latest 252 daily log returns.
3. Implements European Black-Scholes prices and deltas.
4. Recovers each call's implied volatility by root finding.
5. Uses the strike closest to spot as the only calibration quote.
6. Prices the remaining strikes using that single ATM implied volatility.
7. Reports bias, MAE, RMSE and static-arbitrage diagnostics.
8. checks the analytic ATM price with antithetic Monte Carlo simulation.
9. Fully revalues one long 100-share call under 500 historical daily shocks and estimates 97.5% VaR/CVaR.

## Core mathematics

For a European call,

\[
C_{BS}=Se^{-qT}\Phi(d_1)-Ke^{-rT}\Phi(d_2),
\qquad
d_1=\frac{\log(S/K)+(r-q+\tfrac12\sigma^2)T}{\sigma\sqrt T},
\quad d_2=d_1-\sigma\sqrt T.
\]

If \(M_i\) is the market price at strike \(K_i\), the notebook chooses

\[
i^*=\arg\min_i |\log(K_i/S)|
\]

and solves \(C_{BS}(K_{i^*},\widehat\sigma_{ATM})=M_{i^*}\). It then holds \(\widehat\sigma_{ATM}\) fixed and calculates

\[
e_i=C_{BS}(K_i,\widehat\sigma_{ATM})-M_i
\]

at all other strikes. Bias is the mean of \(e_i\), MAE is the mean of \(|e_i|\), and RMSE is the square root of the mean of \(e_i^2\).

For historical daily log-return \(x_i\), the one-contract scenario loss is

\[
L_i=-100\left[V(Se^{x_i},T-1/252)-V(S,T)\right].
\]

VaR is the empirical 97.5% loss quantile. CVaR, also called Expected Shortfall, is the average of the worst 13 losses among 500 scenarios.

## Stored results

The notebook is not one synchronized run: AMZN, META, GOOGL and NVDA are dated 26 August 2026 with 25 September expiry, whereas AAPL and TSLA retain 18 August outputs with 18 September expiry.

| Ticker | Hist. vol | ATM IV | MAE | RMSE | VaR USD | CVaR USD |
|---|---:|---:|---:|---:|---:|---:|
| AMZN | 33.77% | 29.73% | 0.411 | 0.717 | 512.36 | 594.33 |
| META | 38.82% | 36.01% | 0.777 | 1.337 | 1,187.29 | 1,543.23 |
| AAPL | 25.00% | 23.11% | 0.166 | 0.217 | 512.76 | 632.00 |
| GOOGL | 32.27% | 27.51% | 0.802 | 1.958 | 645.14 | 766.69 |
| NVDA | 36.77% | 41.39% | 0.181 | 0.274 | 519.66 | 620.12 |
| TSLA | 46.63% | 35.94% | 0.925 | 1.331 | 1,009.96 | 1,220.64 |

![Stored AMZN validation graph](amzn_validation_original_v5.png)

## Interpretation

- The parity, delta and Monte Carlo checks pass, supporting internal consistency of the Black-Scholes implementation.
- AAPL and NVDA have the smallest stored pricing errors. GOOGL has the largest RMSE, while TSLA has the largest MAE. These are absolute option-dollar errors and are not normalized across ticker scale.
- META has the largest stored one-day VaR and CVaR. CVaR exceeds VaR for every ticker, as expected because it averages losses beyond the VaR threshold.
- Strike-dependent implied volatility is evidence against the single constant-volatility assumption.

## Critical data warning

Every stored ticker run reports zero usable bid-ask midpoints and uses last-price fallback for every retained option. Therefore `inside_bid_ask_pct` and `median_error_in_spreads` are undefined. Last prices can be stale and asynchronous, so the bound, monotonicity and convexity counts are mainly data-quality warnings, not proven executable arbitrage.

## Important limitations

- Calibration to one ATM quote is highly sensitive to that single observation.
- This is cross-strike validation on one date, not out-of-time testing.
- The code fixes `q=0` during VaR revaluation, even if another dividend yield were supplied.
- VaR/CVaR shock only spot; volatility, rates, dividends and liquidity remain frozen.
- Only 13 observations determine CVaR, making it statistically unstable.
- The notebook estimates VaR/CVaR but does not backtest their exception rate.
- Yahoo-listed US equity options are American-style, while the benchmark is European Black-Scholes.
- The function's comments mention a seven-day minimum, but the implemented expiry rule selects at least 30 days.

## Running the notebook

Install `yfinance` in a notebook cell with `%pip install yfinance`, restart the kernel if required, and run all cells in order. A validation call writes a strike-level CSV, a summary CSV and a three-panel PNG:

```python
validate_market_chain(
    ticker="AMZN",
    start="2018-01-01",
    expiry=None,
    r=0.04,
    q=0.00,
    var_confidence=0.975,
    output_dir=".",
)
```

This is a concise educational validation prototype, not a production model approval or regulatory VaR framework.
