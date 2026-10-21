# Order-Book Paper Trading Simulator

> **Status: In progress.** Design is complete; the market-data client is under active development. See the [Roadmap](#roadmap).

A paper-trading engine that simulates **realistic order execution** by filling orders against a live Level-2 order book, instead of filling instantly at the last traded price.

---

## The Problem

Most paper-trading platforms fill your entire order instantly at the displayed price. Real markets don't work that way:

- **Slippage**: a large market order consumes several price levels, so the average fill price is worse than the top-of-book price.
- **Partial fills**: limit orders may only be partially filled if there isn't enough liquidity at that price.
- **Fees**: commissions and exchange fees reduce returns, and strategies that look profitable on paper can lose money once costs are applied.

Because of this, strategies that work in a simulator can fail in real trading. This project models these frictions using **real order-book depth**.

---

## How It Works

```
 Exchange WebSocket            Local Order Book            Fill Simulator
 (L2 depth stream)  ──────▶   (bids / asks by price) ───▶  (walks the book)
        │                            │                            │
        ▼                            ▼                            ▼
   Recorder  ──────────▶   Replay Engine (deterministic)    Portfolio & P&L
   (raw messages to disk)  feeds recorded data back in      (fees applied)
```

### 1. Market Data Client
- Connects to an exchange's public depth stream (initially **Binance**) over WebSocket.
- Keeps a **local order book** by applying incremental updates on top of an initial REST snapshot.
- Detects **sequence gaps** (missed messages) and resynchronises from a fresh snapshot.

### 2. Fill Simulator: "walking the book"
A market **buy** order consumes ask levels starting from the best (lowest) price:

```
Order: BUY 3.0 BTC (market)

Asks:  price     qty
       64,000    1.0   ← fill 1.0 @ 64,000
       64,010    1.5   ← fill 1.5 @ 64,010
       64,025    2.0   ← fill 0.5 @ 64,025

Average fill price = (1.0×64,000 + 1.5×64,010 + 0.5×64,025) / 3.0 = 64,009.17
Slippage vs. best ask = 9.17 (≈ 1.4 bps)
```

Sells work the same way against the bids. Limit orders fill only at or better than their limit price, and partial fills are supported.

### 3. Record & Replay
- Every raw market-data message is **recorded to disk** with its timestamp.
- The **replay engine** feeds recorded data back through the same pipeline, so the same input always produces the same fills.
- This makes the simulator **testable and debuggable**: any bug can be reproduced exactly.

---

## Key Design Decisions

| Decision | Reason |
|---|---|
| **Crypto order books as the data source** | Free US equity feeds provide only top-of-book prices, with no depth. Major crypto exchanges publish full L2 depth for free, which this simulator needs. |
| **`BigDecimal` / integer units for prices and quantities** | Floating-point rounding errors are unacceptable in financial calculations. |
| **Deterministic replay** | Live markets never repeat, so testing against recorded data is the only way to get reproducible results. |
| **Separate layers (ingestion → book → simulator → portfolio)** | Each layer can be tested in isolation, and the data source can be swapped without touching the simulation logic. |
| **Assumption: simulated orders do not affect the market** | Orders are filled against the real book but are never sent to the exchange. This is a known limitation for very large orders. |

---

## Tech Stack

- **Backend:** Java 17+, Java WebSocket client, `CompletableFuture` for async processing
- **Testing:** JUnit 5, replay-based integration tests
- **Frontend (planned):** React + TradingView Lightweight Charts
- **Data:** Binance public market-data API (no account required)

---

## Roadmap

**Phase 1: Market Data**
- [x] System design and architecture
- [ ] WebSocket client for Binance depth stream
- [ ] Local order book with snapshot sync
- [ ] Sequence-gap detection and resync
- [ ] Recording raw messages to disk

**Phase 2: Simulation**
- [ ] Market orders: walk the book with slippage calculation
- [ ] Limit orders with partial fills
- [ ] Deterministic replay engine
- [ ] Unit and replay-based tests

**Phase 3: Realism & Validation**
- [ ] Configurable fee models (maker/taker fees)
- [ ] Simulated order latency
- [ ] Validate simulated fills against actual subsequent trades (error in bps)

**Phase 4: Interface**
- [ ] REST/WebSocket API for placing orders and streaming portfolio updates
- [ ] Web dashboard with live chart, order entry, and fill analytics
- [ ] Trading-behaviour report (stop-loss usage, hold time of winners vs. losers)

---

## Known Limitations
- Simulated orders don't consume liquidity for later orders, and don't move the market.
- Queue position for resting limit orders is approximated, not modelled exactly.
- Currently supports a single exchange and a few trading pairs.

---

## Author
**Annika Garg**, Computer Engineering, National University of Singapore
