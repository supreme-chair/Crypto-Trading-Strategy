# Team 1 - Crypto Trading Strategy
Team Members
Zagrekov Kirill - supreme-chair (GitHub)
Bakalenko Pavel - pavelbakalenko-pixel (GitHub)
Dribnokhod Evgeniy - zemlianin (GitHub)
# Team 3 - Crypto Trading Strategy


## Strategy Overview

### Architecture
Implementation is delivered as an n8n workflow (`workflow.json`) that builds a daily dataset (market + news), runs batched sentiment inference, and performs a rule-based backtest with risk controls, exporting a trade log and summary metrics.

Pipeline (single workflow):

1. Load data
   - 1. Load Market Data — HTTP request to crypto_features_3months.csv
   - 2. Load News Data — HTTP request to crypto_news_3months.csv

2. Parse & merge
   - Parse CSV (Market) — parses market CSV to JSON rows
   - Parse CSV (News) — parses news CSV to JSON rows
   - Group News — groups news titles by date
   - 3. Merge Daily Data — merges daily market row with up to 3 news titles into news_text
     - If no news: "Market is quiet today."

3. Batch sentiment inference (single call)
   - 4. Prepare Batch — creates batch_inputs array from news_text
   - 5. HF Sentiment (Single Batch) — POST to HuggingFace Inference Router:
     distilbert/distilbert-base-uncased-finetuned-sst-2-english

4. Trading logic + metrics
   - 6. Final Logic & Metrics — runs portfolio simulation, builds trade log, computes metrics:
     Sharpe (annualized), Max Drawdown, Win Rate, Profit Factor, Total Return, Final Balance

5. Export
   - 7. Export CSV — exports trade_log.csv / metrics output (depending on workflow setup)

> Notes:
> - The workflow is designed to run end-to-end without a separate Python codebase.
> - Concurrency is not used; sentiment is computed via one batched request.

### Sentiment / “LLM” Metrics Design
Model used for inference is DistilBERT fine-tuned on SST-2 via HuggingFace Inference Router.
The workflow converts the HF output into a signed signal:

- POSITIVE → +score
- NEGATIVE → -score
- otherwise → 0

This signed sentiment is used as the primary “news regime” gate in entries/exits.

### Business Logic Design

Strategy type: rule-based “no-hold” regime with selective entries, plus tight risk controls.

Key mechanics inside 6. Final Logic & Metrics:

#### Portfolio setup
- Universe: derived from unique tickers in market CSV
- Capital allocation: $10,000 / number_of_tickers per ticker wallet
- State per ticker: balance, coin, inPosition, entryPrice, peakPrice, cooldownUntil

#### Risk parameters
- Stop-loss: 1.8%
- Take-profit: 3.0%
- Trailing stop: 1.5%
- Cooldown after a sell: 3 days (reduces churn)

#### Technical inputs (safe parsing)
- RSI (default 50 if missing)
- Optional moving averages (fast/slow) if present in CSV
- Trend regime:
  - If MAs exist: fastMA > slowMA and price > slowMA ⇒ trendUp
  - Else fallback: RSI-based (`rsi >= 52` ⇒ up, rsi <= 48 ⇒ down)

#### “No HOLD” policy (important)
Default behavior is explicitly:
- If flat → decision is "sell" (stay in cash)
- If in position → decision is "buy" (stay long)

Trades happen only on state changes (flat→long or long→flat).

#### Entry rules (only when flat & not in cooldown)
Two entry modes:

1) Trend-following (high selectivity)
- trendUp AND signedSent >= 0.75 AND 40 <= RSI <= 65

2) Mean-reversion (oversold bounce)
- RSI <= 30 AND signedSent >= 0.10 (explicitly avoids negative sentiment)

#### Exit rules (only when in position)
Exit triggers are checked in this order:

1) Stop-loss hit
2) Trailing stop hit (only after price has moved > ~1% above entry)
3) Take-profit hit
4) Strong negative sentiment: signedSent <= -0.75
5) Technical exit: trendDown OR (RSI >= 70 AND signedSent <= 0.10)

After selling: ticker enters cooldown for 3 days.

### Decision Flowchart

```mermaid
graph TD
    Start[Load market and news CSV] --> Parse[Parse CSV files]
    Parse --> Merge[Merge daily rows with news]
    Merge --> Batch[Prepare batch inputs]
    Batch --> HF[HF sentiment model SST2 batch request]
    HF --> Logic[Portfolio simulation and rules]
    Logic --> Entry{Flat and not in cooldown}
    Entry -->|Yes| EntryRules{Entry condition met}
    EntryRules -->|Yes| Buy[BUY enter position]
    EntryRules -->|No| StayFlat[SELL stay flat]
    Entry -->|No| InPos{In position}
    InPos -->|Yes| ExitRules{Exit trigger met}
    ExitRules -->|Yes| Sell[SELL exit and set cooldown]
    ExitRules -->|No| HoldLong[BUY keep position]
    Logic --> Metrics[Compute metrics]
    Metrics --> Export[Export CSV outputs]
