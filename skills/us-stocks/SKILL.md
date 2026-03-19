---
name: us-stocks
description: US stock market research — quotes, fundamentals, news, technical signals, and buy/sell analysis.
homepage: https://finance.yahoo.com
metadata:
  {
    "openclaw":
      {
        "emoji": "📈",
        "requires": { "bins": ["curl", "jq"] },
      },
  }
---

# US Stock Market Research

Research US equities and get buy/sell recommendations using free public APIs. No API key required for core functionality. An optional Alpha Vantage key unlocks technical indicators and deeper fundamentals.

## Quick Quote (Yahoo Finance — no key needed)

```bash
SYMBOL=AAPL
curl -s "https://query1.finance.yahoo.com/v8/finance/chart/$SYMBOL?interval=1d&range=1d" \
  | jq '.chart.result[0].meta | {symbol, regularMarketPrice, previousClose, regularMarketChangePercent, regularMarketVolume, marketCap, fiftyTwoWeekHigh, fiftyTwoWeekLow}'
```

Key fields returned:
- `regularMarketPrice` — current price
- `regularMarketChangePercent` — % change today
- `fiftyTwoWeekHigh` / `fiftyTwoWeekLow` — 52-week range
- `marketCap` — market capitalisation

## Company Fundamentals (Yahoo Finance)

```bash
SYMBOL=MSFT
curl -s "https://query1.finance.yahoo.com/v10/finance/quoteSummary/$SYMBOL?modules=defaultKeyStatistics,financialData,summaryDetail" \
  | jq '.quoteSummary.result[0] | {
      forwardPE: .defaultKeyStatistics.forwardPE.raw,
      trailingPE: .defaultKeyStatistics.trailingEps.raw,
      priceToBook: .defaultKeyStatistics.priceToBook.raw,
      debtToEquity: .financialData.debtToEquity.raw,
      returnOnEquity: .financialData.returnOnEquity.raw,
      revenueGrowth: .financialData.revenueGrowth.raw,
      grossMargins: .financialData.grossMargins.raw,
      dividendYield: .summaryDetail.dividendYield.raw,
      beta: .summaryDetail.beta.raw
    }'
```

## Historical Prices (last 30 days)

```bash
SYMBOL=NVDA
curl -s "https://query1.finance.yahoo.com/v8/finance/chart/$SYMBOL?interval=1d&range=1mo" \
  | jq '[.chart.result[0] | {
      timestamps: .timestamp,
      closes: .indicators.quote[0].close
    } | .timestamps as $t | .closes as $c | range($t|length) | {date: ($t[.] | todate), close: $c[.]}]'
```

## Analyst Recommendations (Yahoo Finance)

```bash
SYMBOL=TSLA
curl -s "https://query1.finance.yahoo.com/v10/finance/quoteSummary/$SYMBOL?modules=recommendationTrend,upgradeDowngradeHistory" \
  | jq '.quoteSummary.result[0].recommendationTrend.trend[0]'
# Returns: strongBuy, buy, hold, sell, strongSell counts from analysts
```

## Recent News

```bash
SYMBOL=AMZN
curl -s "https://query1.finance.yahoo.com/v1/finance/search?q=$SYMBOL&newsCount=10&quotesCount=0" \
  | jq '.news[] | {title, publisher, providerPublishTime: (.providerPublishTime | todate), link}'
```

## Market Movers (top gainers / losers)

```bash
# Top gainers
curl -s "https://query1.finance.yahoo.com/v1/finance/screener/predefined/saved?scrIds=day_gainers&count=10" \
  | jq '.finance.result[0].quotes[] | {symbol, shortName, regularMarketChangePercent, regularMarketPrice}'

# Top losers
curl -s "https://query1.finance.yahoo.com/v1/finance/screener/predefined/saved?scrIds=day_losers&count=10" \
  | jq '.finance.result[0].quotes[] | {symbol, shortName, regularMarketChangePercent, regularMarketPrice}'

# Most active
curl -s "https://query1.finance.yahoo.com/v1/finance/screener/predefined/saved?scrIds=most_actives&count=10" \
  | jq '.finance.result[0].quotes[] | {symbol, shortName, regularMarketVolume, regularMarketPrice}'
```

## Index Snapshot (SPY, QQQ, DIA)

```bash
for SYMBOL in SPY QQQ DIA IWM; do
  curl -s "https://query1.finance.yahoo.com/v8/finance/chart/$SYMBOL?interval=1d&range=1d" \
    | jq --arg s "$SYMBOL" '.chart.result[0].meta | [$s, .regularMarketPrice, .regularMarketChangePercent] | @tsv'
done
```

## Sector Performance

```bash
# Use sector ETFs as proxies
declare -A SECTORS=(
  [Technology]=XLK [Health]=XLV [Finance]=XLF [Energy]=XLE
  [Consumer]=XLY [Utilities]=XLU [Industrials]=XLI [Materials]=XLB
)
for NAME in "${!SECTORS[@]}"; do
  ETF="${SECTORS[$NAME]}"
  CHANGE=$(curl -s "https://query1.finance.yahoo.com/v8/finance/chart/$ETF?interval=1d&range=1d" \
    | jq '.chart.result[0].meta.regularMarketChangePercent')
  echo "$NAME ($ETF): $CHANGE%"
done
```

## Technical Indicators (Alpha Vantage — optional free key)

Sign up at https://www.alphavantage.co/support/#api-key (free: 25 req/day).

```bash
AV_KEY=your_key_here   # or: export ALPHA_VANTAGE_KEY=...
SYMBOL=AAPL

# RSI (14-day)
curl -s "https://www.alphavantage.co/query?function=RSI&symbol=$SYMBOL&interval=daily&time_period=14&series_type=close&apikey=$AV_KEY" \
  | jq '.Technical Analysis: RSI | to_entries | first | {date: .key, rsi: .value.RSI}'

# MACD
curl -s "https://www.alphavantage.co/query?function=MACD&symbol=$SYMBOL&interval=daily&series_type=close&apikey=$AV_KEY" \
  | jq '."Technical Analysis: MACD" | to_entries | first | {date: .key, macd: .value.MACD, signal: .value.MACD_Signal, hist: .value.MACD_Hist}'

# 50-day & 200-day SMA
for PERIOD in 50 200; do
  curl -s "https://www.alphavantage.co/query?function=SMA&symbol=$SYMBOL&interval=daily&time_period=$PERIOD&series_type=close&apikey=$AV_KEY" \
    | jq --arg p "$PERIOD" '."Technical Analysis: SMA" | to_entries | first | {date: .key, ("sma_"+$p): .value.SMA}'
done
```

## Buy / Sell Signal Analysis Framework

When asked for a buy or sell recommendation, gather the following and apply this framework:

### Step 1 — Collect data

```bash
SYMBOL=AAPL

# 1. Quote + 52-week range
QUOTE=$(curl -s "https://query1.finance.yahoo.com/v8/finance/chart/$SYMBOL?interval=1d&range=1d" | jq '.chart.result[0].meta')

# 2. Fundamentals
FUND=$(curl -s "https://query1.finance.yahoo.com/v10/finance/quoteSummary/$SYMBOL?modules=defaultKeyStatistics,financialData,summaryDetail" | jq '.quoteSummary.result[0]')

# 3. Analyst consensus
RECO=$(curl -s "https://query1.finance.yahoo.com/v10/finance/quoteSummary/$SYMBOL?modules=recommendationTrend" \
  | jq '.quoteSummary.result[0].recommendationTrend.trend[0]')

# 4. Recent news headlines (sentiment check)
NEWS=$(curl -s "https://query1.finance.yahoo.com/v1/finance/search?q=$SYMBOL&newsCount=5&quotesCount=0" \
  | jq '[.news[].title]')
```

### Step 2 — Apply scoring rubric

Score each signal (+1 bullish, -1 bearish, 0 neutral):

| Signal | Bullish condition | Bearish condition |
|--------|------------------|------------------|
| Price vs 52-week range | Within 10% of high | Within 10% of low |
| P/E ratio | Below sector average | Above 2× sector avg |
| Revenue growth | >10% YoY | Negative |
| Analyst consensus | Strong buy / buy majority | Sell / strong sell majority |
| Debt-to-equity | <1.0 | >2.0 |
| Return on equity | >15% | <5% |
| News sentiment | Mostly positive | Mostly negative |

### Step 3 — Render verdict

- **Score ≥ +3**: BUY signal — outline entry price, stop-loss (e.g. 8% below entry), and target (e.g. 15–20% upside)
- **Score -2 to +2**: HOLD / NEUTRAL — explain key risks and catalysts to watch
- **Score ≤ -3**: SELL / AVOID — explain downside thesis

Always include:
- Disclaimer that this is research output, not financial advice
- Key risks specific to the company (competitive, regulatory, macro)
- Suggested position sizing note (e.g. diversify, use limit orders)

## Compare Two Stocks

```bash
for SYMBOL in META GOOG; do
  echo "=== $SYMBOL ==="
  curl -s "https://query1.finance.yahoo.com/v10/finance/quoteSummary/$SYMBOL?modules=defaultKeyStatistics,financialData" \
    | jq '.quoteSummary.result[0] | {
        forwardPE: .defaultKeyStatistics.forwardPE.raw,
        priceToBook: .defaultKeyStatistics.priceToBook.raw,
        revenueGrowth: .financialData.revenueGrowth.raw,
        returnOnEquity: .financialData.returnOnEquity.raw,
        grossMargins: .financialData.grossMargins.raw
      }'
done
```

## Portfolio Snapshot

Given a list of holdings, print current values:

```bash
HOLDINGS=("AAPL:50" "MSFT:30" "NVDA:20" "AMZN:15")
TOTAL=0
for HOLDING in "${HOLDINGS[@]}"; do
  SYMBOL="${HOLDING%%:*}"
  SHARES="${HOLDING##*:}"
  PRICE=$(curl -s "https://query1.finance.yahoo.com/v8/finance/chart/$SYMBOL?interval=1d&range=1d" \
    | jq '.chart.result[0].meta.regularMarketPrice')
  VALUE=$(echo "$PRICE * $SHARES" | bc)
  echo "$SYMBOL: $SHARES shares @ \$$PRICE = \$$VALUE"
done
```

## Notes

- Yahoo Finance endpoints are unofficial and may change — prefer `query1.finance.yahoo.com` over `query2`
- Rate limit: be conservative, ~10 req/min to avoid 429s
- Alpha Vantage free tier: 25 requests/day, 5/minute
- Market hours: 9:30 AM – 4:00 PM ET (Mon–Fri); pre/post-market data available with `includePrePost=true`
- Always confirm ticker symbols — e.g. `BRK.B` not `BRK-B` for Yahoo Finance
- This skill is for research only. Always remind the user that output is not financial advice.
