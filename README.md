# Money Matrix — Methodology Document
**Strategy:** EMA Crossover (20/50) + RSI(14) + Bollinger Band Width  
**Period:** May 2019 – May 2026 | **Capital:** $1,000,000

---

## 1. The Alpha Hypothesis

My core belief is that stock prices exhibit short-to-medium term momentum - stocks that are trending up tend to keep going up, until they get stretched too far and reverse. I wanted to exploit this by entering a position when a trend is just forming, not after it has already run too far.

To do this I combined three indicators that each answer a different question:

| Indicator | Question it answers | Why I chose it |
|-----------|-------------------|----------------|
| EMA Crossover (20/50) | Is there an uptrend forming? | EMAs smooth noise better than simple MAs; crossover confirms direction |
| RSI (14-day) | Has the stock already run too far? | Prevents entering overbought positions near short-term peaks |
| Bollinger Band Width | Is there enough volatility to trade? | Filters out "squeeze" phases where EMA crossovers generate false signals |

**Combined signal logic:**
```
BUY  (+1) : EMA_20 > EMA_50  AND  RSI < 65  AND  BB_width > 0.04
SELL (-1) : EMA_20 ≤ EMA_50   OR  RSI > 65
HOLD  (0) : everything else
```

All three must agree for a BUY. Any single sell condition is enough to exit.

I coded all three indicators from scratch without ta-lib. The guide said to understand why they generate signals, not just use a library.

---

## 2. Parameter Choices

| Parameter | Value | Reasoning |
|-----------|-------|-----------|
| EMA Fast | 20 days | ~1 calendar month; responsive without excessive noise |
| EMA Slow | 50 days | ~2.5 months; represents medium-term trend direction |
| RSI Period | 14 days | Wilder's original recommendation; stable and widely used |
| RSI Overbought | 65 | Slightly conservative vs classic 70; reduces late entries |
| BB Window | 20 days | Matches EMA fast window; one month of volatility context |
| BB Width Min | 0.04 (4%) | Below this = squeeze = signals unreliable |
| Rebalancing | Monthly | Low transaction cost; realistic without daily infrastructure |
| Transaction Cost | 10 bps | Conservative estimate of commission + slippage for large-caps |

I chose 65 instead of the classic 70 for RSI because I wanted to be a bit more conservative about entering overbought situations. In hindsight (see Section 4), this may have been too aggressive a filter for a bull market period.

---

## 3. Backtesting Design

The backtest runs monthly from June 2019 to May 2026 — 84 months total.

**Key design decisions:**

**No look-ahead bias:** The signal from end of month M drives trades executed at the start of month M+1. Implemented via a 1-month `.shift(1)` on the signal DataFrame. Without this, the backtest would peek at future prices and the results would be meaningless.

**Equal weighting:** Capital split equally among all BUY stocks each month. Average of 4.5 stocks held per month. 6 months the portfolio was 100% cash (no stocks had a buy signal).

**Transaction costs:** 0.1% deducted on every trade (both buys and sells). With 105.2% average monthly turnover, this cost drag is significant.

**Results:**

| Metric | Strategy | Benchmark |
|--------|----------|-----------|
| Annualised Return | 13.92% | 26.75% |
| Annualised Volatility | 19.34% | 17.80% |
| Sharpe Ratio | 0.51 | 1.28 |
| Max Drawdown | -36.86% | -21.88% |
| Calmar Ratio | 0.38 | 1.22 |
| Final Value | $2,490,680 | $5,256,152 |

---

## 4. Robustness Checks (Phase 4)

### 4.1 Train/Test Split

| Period | Ann. Return | Ann. Vol | Sharpe | Max DD |
|--------|-------------|----------|--------|--------|
| Train (2019–2024, 5yr) | 10.7% | 20.7% | 0.32 | -36.9% |
| Test (2024–2026, 2yr) | 16.7% | 14.8% | 0.86 | -19.4% |

The test Sharpe (0.86) is higher than the train Sharpe (0.32). This is the opposite of what an overfit strategy would show — an overfit strategy degrades out-of-sample. The train period's poor numbers are partly explained by the COVID crash of 2020, which caused a -36.9% drawdown.

### 4.2 Parameter Sensitivity

| EMA Windows | Ann. Return | Sharpe |
|-------------|-------------|--------|
| (15, 40) | 15.3% | 0.59 |
| (18, 45) | 15.7% | 0.61 |
| **(20, 50) — base** | **13.9%** | **0.51** |
| (22, 55) | 14.8% | 0.57 |
| (25, 60) | 17.8% | 0.69 |

Sharpe std across variants: **0.056** — the strategy does not collapse when parameters shift slightly. This is a sign of genuine robustness rather than curve-fitting.

### 4.3 Look-Ahead Bias Prevention

Every signal is generated using only data available up to and including day X to make trades on day X+1. The 1-month shift in the backtesting engine enforces this strictly.

---

## 5. Honest Assessment — Why the Strategy Underperformed

The strategy returned 149% vs the benchmark's 426% over 7 years. The -10.69% annual alpha is significant and worth explaining honestly.

**What went wrong:**

The 2019–2026 period was dominated by a handful of stocks on massive momentum runs — especially NVDA (~3000% over 7 years), MSFT, and AAPL. These stocks stayed "overbought" (RSI > 65) for months at a time because the market kept pushing them higher regardless of short-term overextension.I used 65 instead of the classic 70 to be slightly more conservative — meaning I exit positions earlier before they become fully overbought.

My RSI filter kept exiting these positions every time they looked overbought — but they just kept going. The filter that was meant to protect against buying at peaks ended up making me miss most of the biggest winners in the universe.

The 105.2% monthly turnover also hurt — constantly switching in and out of positions while paying 0.1% each time compounded into meaningful drag.

**What the strategy did right:**

- Positive out-of-sample Sharpe (0.86) — not overfit to the training period
- Low parameter sensitivity (Sharpe std = 0.056) — robust to small changes
- 59% win rate with 1.19x profit factor — more winning months than losing
- All indicators coded manually with clear mathematical rationale

**What I would try next:**

- Relax or remove the RSI exit filter in strong uptrends — momentum stocks need to be held
- Add a market regime filter: only use RSI as an exit in sideways/bear markets
- Use volatility-based position sizing instead of equal weight
- Add stop-losses to reduce the -36.86% max drawdown
