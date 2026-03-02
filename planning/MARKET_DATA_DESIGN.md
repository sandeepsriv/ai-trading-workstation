# Market Data Backend — Design Document

Complete design for the AI Trading Workstation market data subsystem. This document covers every module, interface, data flow, and integration point with code snippets ready for implementation.

---

## Table of Contents

1. [Architecture Overview](#1-architecture-overview)
2. [File Structure](#2-file-structure)
3. [Data Model — `models.py`](#3-data-model--modelspy)
4. [Price Cache — `cache.py`](#4-price-cache--cachepy)
5. [Abstract Interface — `interface.py`](#5-abstract-interface--interfacepy)
6. [Seed Prices & Parameters — `seed_prices.py`](#6-seed-prices--parameters--seed_pricespy)
7. [GBM Simulator — `simulator.py`](#7-gbm-simulator--simulatorpy)
8. [Massive API Client — `massive_client.py`](#8-massive-api-client--massive_clientpy)
9. [Factory — `factory.py`](#9-factory--factorypy)
10. [SSE Streaming — `stream.py`](#10-sse-streaming--streampy)
11. [Module Exports — `__init__.py`](#11-module-exports--__init__py)
12. [FastAPI Lifecycle Integration](#12-fastapi-lifecycle-integration)
13. [Watchlist Coordination](#13-watchlist-coordination)
14. [Data Flow Diagrams](#14-data-flow-diagrams)
15. [Testing Strategy](#15-testing-strategy)
16. [Error Handling & Edge Cases](#16-error-handling--edge-cases)
17. [Configuration Reference](#17-configuration-reference)

---

## 1. Architecture Overview

The market data subsystem follows a **strategy pattern** with a shared cache:

```
                 ┌─────────────────────────────────┐
                 │     Environment Variable         │
                 │     MASSIVE_API_KEY              │
                 └──────────┬──────────────────────┘
                            │
                     ┌──────▼──────┐
                     │   Factory   │
                     │ factory.py  │
                     └──────┬──────┘
                            │
              ┌─────────────┴─────────────┐
              │                           │
   ┌──────────▼──────────┐    ┌───────────▼───────────┐
   │  SimulatorDataSource │    │  MassiveDataSource     │
   │  (GBM simulation)   │    │  (Polygon.io REST)     │
   │  500ms ticks         │    │  15s poll interval     │
   └──────────┬──────────┘    └───────────┬───────────┘
              │                           │
              │  Both implement           │
              │  MarketDataSource ABC     │
              │                           │
              └─────────────┬─────────────┘
                            │ writes
                    ┌───────▼────────┐
                    │   PriceCache   │
                    │  (thread-safe) │
                    └───────┬────────┘
                            │ reads
              ┌─────────────┼─────────────┐
              │             │             │
     ┌────────▼──┐  ┌───────▼──┐  ┌───────▼────────┐
     │ SSE Stream│  │Portfolio │  │Trade Execution │
     │ /api/     │  │Valuation │  │POST /api/      │
     │ stream/   │  │GET /api/ │  │portfolio/trade │
     │ prices    │  │portfolio │  │                │
     └───────────┘  └──────────┘  └────────────────┘
```

### Core Principles

- **One writer, many readers**: Only one data source (simulator *or* Massive) writes to the cache at any time. Multiple consumers read from it concurrently.
- **Push model**: Data sources push updates into the cache on their own schedule. Consumers poll the cache. There is no callback or observer mechanism — the version counter is sufficient.
- **Source-agnostic downstream**: SSE streaming, portfolio valuation, and trade execution never know which data source is active. They only interact with `PriceCache`.
- **Thread safety**: `PriceCache` uses `threading.Lock` because the Massive client runs synchronous HTTP calls in `asyncio.to_thread()`, which operates in a real OS thread.

---

## 2. File Structure

```
backend/
  app/
    market/
      __init__.py             # Public API re-exports
      models.py               # PriceUpdate dataclass
      cache.py                # PriceCache (thread-safe in-memory store)
      interface.py            # MarketDataSource ABC
      seed_prices.py          # Seed prices, GBM params, correlation groups
      simulator.py            # GBMSimulator + SimulatorDataSource
      massive_client.py       # MassiveDataSource (Polygon.io REST poller)
      factory.py              # create_market_data_source()
      stream.py               # SSE endpoint (FastAPI router factory)
```

Each file has a single responsibility. The `__init__.py` re-exports the public API so downstream code imports from `app.market` without reaching into submodules.

---

## 3. Data Model — `models.py`

`PriceUpdate` is the **only data structure** that leaves the market data layer. Every downstream consumer — SSE streaming, portfolio valuation, trade execution — works exclusively with this type.

```python
from __future__ import annotations

import time
from dataclasses import dataclass, field


@dataclass(frozen=True, slots=True)
class PriceUpdate:
    """Immutable snapshot of a single ticker's price at a point in time.

    Created by PriceCache.update() whenever a data source reports a new price.
    Frozen and slotted for safety and memory efficiency.
    """

    ticker: str
    price: float
    previous_price: float
    timestamp: float = field(default_factory=time.time)  # Unix seconds

    @property
    def change(self) -> float:
        """Absolute price change from previous update."""
        return round(self.price - self.previous_price, 4)

    @property
    def change_percent(self) -> float:
        """Percentage change from previous update."""
        if self.previous_price == 0:
            return 0.0
        return round(
            (self.price - self.previous_price) / self.previous_price * 100, 4
        )

    @property
    def direction(self) -> str:
        """'up', 'down', or 'flat'."""
        if self.price > self.previous_price:
            return "up"
        elif self.price < self.previous_price:
            return "down"
        return "flat"

    def to_dict(self) -> dict:
        """Serialize for JSON / SSE transmission."""
        return {
            "ticker": self.ticker,
            "price": self.price,
            "previous_price": self.previous_price,
            "timestamp": self.timestamp,
            "change": self.change,
            "change_percent": self.change_percent,
            "direction": self.direction,
        }
```

### Design Decisions

| Decision | Rationale |
|----------|-----------|
| `frozen=True` | Price updates are immutable value objects. Once created they never change, making them safe to share across async tasks. |
| `slots=True` | Minor memory optimization — the system creates many of these per second. |
| Computed properties | `change`, `direction`, and `change_percent` derive from `price` and `previous_price`, so they can never be inconsistent. |
| `to_dict()` | Single serialization point used by both SSE events and REST API responses. Keeps JSON shape centralized. |
| `timestamp` default | Uses `time.time()` at creation if not provided, which is correct for the simulator. The Massive client passes explicit timestamps from the API response. |

### Example Usage

```python
# Created by the cache, not directly
update = PriceUpdate(ticker="AAPL", price=191.50, previous_price=190.42)

update.change          # 1.08
update.change_percent  # 0.5672
update.direction       # "up"
update.to_dict()       # {"ticker": "AAPL", "price": 191.50, ...}
```

---

## 4. Price Cache — `cache.py`

The price cache is the **central data hub**. Data sources write to it; SSE streaming and portfolio valuation read from it.

```python
from __future__ import annotations

import time
from threading import Lock

from .models import PriceUpdate


class PriceCache:
    """Thread-safe in-memory cache of the latest price for each ticker.

    Writers: SimulatorDataSource or MassiveDataSource (one at a time).
    Readers: SSE streaming endpoint, portfolio valuation, trade execution.

    Uses a monotonically increasing version counter for efficient SSE
    change detection — consumers compare their last-seen version to avoid
    sending redundant events.
    """

    def __init__(self) -> None:
        self._prices: dict[str, PriceUpdate] = {}
        self._lock = Lock()
        self._version: int = 0

    def update(self, ticker: str, price: float, timestamp: float | None = None) -> PriceUpdate:
        """Record a new price for a ticker. Returns the created PriceUpdate.

        Automatically computes direction and change from the previous price.
        First update for a ticker: previous_price == price (direction='flat').
        """
        with self._lock:
            ts = timestamp or time.time()
            prev = self._prices.get(ticker)
            previous_price = prev.price if prev else price

            update = PriceUpdate(
                ticker=ticker,
                price=round(price, 2),
                previous_price=round(previous_price, 2),
                timestamp=ts,
            )
            self._prices[ticker] = update
            self._version += 1
            return update

    def get(self, ticker: str) -> PriceUpdate | None:
        """Get the latest price for a single ticker, or None if unknown."""
        with self._lock:
            return self._prices.get(ticker)

    def get_all(self) -> dict[str, PriceUpdate]:
        """Snapshot of all current prices. Returns a shallow copy."""
        with self._lock:
            return dict(self._prices)

    def get_price(self, ticker: str) -> float | None:
        """Convenience: get just the price float, or None."""
        update = self.get(ticker)
        return update.price if update else None

    def remove(self, ticker: str) -> None:
        """Remove a ticker from the cache."""
        with self._lock:
            self._prices.pop(ticker, None)

    @property
    def version(self) -> int:
        """Current version counter. Used for SSE change detection.

        Note: Read without lock. On CPython, int reads are atomic due
        to the GIL. This is intentional — version is only used for
        "has anything changed" polling, not for correctness-critical logic.
        """
        return self._version

    def __len__(self) -> int:
        with self._lock:
            return len(self._prices)

    def __contains__(self, ticker: str) -> bool:
        with self._lock:
            return ticker in self._prices
```

### Version Counter for SSE

The SSE streaming loop polls the cache every ~500ms. Without a version counter, it would serialize and send all prices on every tick even when nothing has changed (e.g., Massive API only updates every 15s). The version counter skips redundant sends:

```python
last_version = -1
while True:
    if price_cache.version != last_version:
        last_version = price_cache.version
        yield format_sse(price_cache.get_all())
    await asyncio.sleep(0.5)
```

### Why `threading.Lock` Instead of `asyncio.Lock`

The Massive client's synchronous `get_snapshot_all()` runs in `asyncio.to_thread()`, which operates in a **real OS thread**. `asyncio.Lock` only protects against concurrent coroutines on the same event loop — it cannot protect against access from a separate thread. `threading.Lock` works correctly from both sync threads and the async event loop.

### Memory Characteristics

- O(n) where n = number of active tickers (typically 10-50)
- Only stores the **latest** price per ticker — no history accumulation
- Each `PriceUpdate` is ~120 bytes (slotted dataclass)
- Total cache memory: negligible even at 100 tickers

---

## 5. Abstract Interface — `interface.py`

The contract that both data sources implement. Downstream code programs against this ABC, never against a concrete implementation.

```python
from __future__ import annotations

from abc import ABC, abstractmethod


class MarketDataSource(ABC):
    """Contract for market data providers.

    Implementations push price updates into a shared PriceCache on their own
    schedule. Downstream code never calls the data source directly for prices —
    it reads from the cache.

    Lifecycle:
        source = create_market_data_source(cache)
        await source.start(["AAPL", "GOOGL", ...])
        # ... app runs ...
        await source.add_ticker("TSLA")
        await source.remove_ticker("GOOGL")
        # ... app shutting down ...
        await source.stop()
    """

    @abstractmethod
    async def start(self, tickers: list[str]) -> None:
        """Begin producing price updates for the given tickers.

        Starts a background task that periodically writes to the PriceCache.
        Must be called exactly once. Calling start() twice is undefined.
        """

    @abstractmethod
    async def stop(self) -> None:
        """Stop the background task and release resources.

        Safe to call multiple times. After stop(), the source will not
        write to the cache again.
        """

    @abstractmethod
    async def add_ticker(self, ticker: str) -> None:
        """Add a ticker to the active set. No-op if already present.

        The next update cycle will include this ticker.
        """

    @abstractmethod
    async def remove_ticker(self, ticker: str) -> None:
        """Remove a ticker from the active set. No-op if not present.

        Also removes the ticker from the PriceCache.
        """

    @abstractmethod
    def get_tickers(self) -> list[str]:
        """Return the current list of actively tracked tickers."""
```

### Why the Source Writes to the Cache (Push Model)

This decouples timing. The simulator ticks at 500ms, Massive polls at 15s, but SSE always reads from the cache at its own 500ms cadence. The SSE layer does not need to know which data source is active or what its update interval is.

### Why All Methods Are `async`

Even though the simulator's `add_ticker()` is synchronous in practice, the interface uses `async` uniformly because:
- The Massive client may need to perform I/O on add/remove (e.g., an immediate poll)
- Consumers use `await` uniformly without type-checking the concrete class
- Cost of `async` on a synchronous operation is negligible

---

## 6. Seed Prices & Parameters — `seed_prices.py`

Pure constants — no logic, no imports beyond stdlib. Shared by the simulator (for initial prices and GBM parameters) and potentially by the Massive client (as fallback prices before the first API response).

```python
"""Seed prices, GBM parameters, and correlation structure for the market simulator."""

# Realistic starting prices for the 10 default watchlist tickers
SEED_PRICES: dict[str, float] = {
    "AAPL": 190.00,
    "GOOGL": 175.00,
    "MSFT": 420.00,
    "AMZN": 185.00,
    "TSLA": 250.00,
    "NVDA": 800.00,
    "META": 500.00,
    "JPM": 195.00,
    "V": 280.00,
    "NFLX": 600.00,
}

# Per-ticker GBM parameters
# sigma: annualized volatility (higher = more price movement per tick)
# mu: annualized drift (expected return direction)
TICKER_PARAMS: dict[str, dict[str, float]] = {
    "AAPL":  {"sigma": 0.22, "mu": 0.05},
    "GOOGL": {"sigma": 0.25, "mu": 0.05},
    "MSFT":  {"sigma": 0.20, "mu": 0.05},
    "AMZN":  {"sigma": 0.28, "mu": 0.05},
    "TSLA":  {"sigma": 0.50, "mu": 0.03},   # High volatility
    "NVDA":  {"sigma": 0.40, "mu": 0.08},   # High volatility, strong drift
    "META":  {"sigma": 0.30, "mu": 0.05},
    "JPM":   {"sigma": 0.18, "mu": 0.04},   # Low volatility (bank)
    "V":     {"sigma": 0.17, "mu": 0.04},   # Low volatility (payments)
    "NFLX":  {"sigma": 0.35, "mu": 0.05},
}

# Default parameters for dynamically added tickers not in the list above
DEFAULT_PARAMS: dict[str, float] = {"sigma": 0.25, "mu": 0.05}

# Correlation groups for Cholesky decomposition
CORRELATION_GROUPS: dict[str, set[str]] = {
    "tech": {"AAPL", "GOOGL", "MSFT", "AMZN", "META", "NVDA", "NFLX"},
    "finance": {"JPM", "V"},
}

# Correlation coefficients
INTRA_TECH_CORR = 0.6       # Tech stocks move together
INTRA_FINANCE_CORR = 0.5    # Finance stocks move together
CROSS_GROUP_CORR = 0.3      # Cross-sector baseline
TSLA_CORR = 0.3             # TSLA does its own thing
```

### Parameter Rationale

| Ticker | sigma | Why |
|--------|-------|-----|
| TSLA | 0.50 | Historically the most volatile major stock. Produces visually dramatic price action. |
| NVDA | 0.40 | High volatility from AI/semiconductor boom. Strong positive drift (0.08) for visual uptrend. |
| V, JPM | 0.17-0.18 | Financials are typically less volatile. Produces calm, steady price paths. |
| MSFT | 0.20 | Blue-chip stability. Lowest tech volatility in the default set. |
| NFLX | 0.35 | Growth stock with earnings-driven swings. |

### Correlation Structure

The correlation matrix enables **correlated moves** across tickers. When the "market" moves, related stocks move together:

```
         AAPL  GOOGL  MSFT  AMZN  TSLA  NVDA  META  JPM   V    NFLX
AAPL     1.0   0.6    0.6   0.6   0.3   0.6   0.6   0.3   0.3  0.6
GOOGL    0.6   1.0    0.6   0.6   0.3   0.6   0.6   0.3   0.3  0.6
MSFT     0.6   0.6    1.0   0.6   0.3   0.6   0.6   0.3   0.3  0.6
AMZN     0.6   0.6    0.6   1.0   0.3   0.6   0.6   0.3   0.3  0.6
TSLA     0.3   0.3    0.3   0.3   1.0   0.3   0.3   0.3   0.3  0.3
NVDA     0.6   0.6    0.6   0.6   0.3   1.0   0.6   0.3   0.3  0.6
META     0.6   0.6    0.6   0.6   0.3   0.6   1.0   0.3   0.3  0.6
JPM      0.3   0.3    0.3   0.3   0.3   0.3   0.3   1.0   0.5  0.3
V        0.3   0.3    0.3   0.3   0.3   0.3   0.3   0.5   1.0  0.3
NFLX     0.6   0.6    0.6   0.6   0.3   0.6   0.6   0.3   0.3  1.0
```

This matrix is always positive definite for these correlation values, which is required for Cholesky decomposition.

---

## 7. GBM Simulator — `simulator.py`

This file contains two classes:
- **`GBMSimulator`**: Pure math engine. Stateful — holds current prices and advances them one step at a time.
- **`SimulatorDataSource`**: The `MarketDataSource` implementation that wraps `GBMSimulator` in an async loop and writes to `PriceCache`.

### 7.1 GBMSimulator — The Math Engine

#### Mathematical Foundation

Geometric Brownian Motion models stock prices as:

```
S(t+dt) = S(t) * exp((mu - 0.5 * sigma^2) * dt + sigma * sqrt(dt) * Z)
```

Where:
- `S(t)` = current price
- `mu` = annualized drift (expected return)
- `sigma` = annualized volatility
- `dt` = time step as fraction of a trading year
- `Z` = standard normal random variable (N(0,1))

The `exp()` function guarantees prices can **never go negative** — this is a fundamental property of GBM.

#### Time Step Calculation

For 500ms update intervals:

```
Trading seconds per year = 252 days * 6.5 hours/day * 3600 sec/hour = 5,896,800
dt = 0.5 / 5,896,800 ≈ 8.48e-8
```

This tiny `dt` produces sub-cent moves per tick that accumulate naturally over time. With sigma=0.22 (AAPL), a single tick moves the price by roughly:

```
sigma * sqrt(dt) * Z ≈ 0.22 * sqrt(8.48e-8) * Z ≈ 0.000064 * Z
For AAPL at $190: ~$0.01 per tick
```

Over a full simulated trading day (23,400 ticks at 2/second), prices drift and diffuse realistically.

#### Correlated Moves via Cholesky Decomposition

To make stocks move together (e.g., tech selloff), we need correlated random draws. The Cholesky decomposition transforms independent standard normals into correlated ones:

```
L = cholesky(CorrelationMatrix)   # Lower triangular matrix
Z_correlated = L @ Z_independent  # Matrix multiplication
```

The resulting `Z_correlated` vector has the exact correlation structure specified by the matrix.

#### Random Shock Events

Every step, each ticker has a 0.1% chance of a "shock event" — a sudden 2-5% move up or down. With 10 tickers at 2 ticks/second, expect an event somewhere roughly every 50 seconds. This adds visual drama to the dashboard.

```python
if random.random() < 0.001:
    shock = random.uniform(0.02, 0.05) * random.choice([-1, 1])
    price *= (1 + shock)
```

#### Implementation

```python
from __future__ import annotations

import asyncio
import logging
import math
import random

import numpy as np

from .cache import PriceCache
from .interface import MarketDataSource
from .seed_prices import (
    CORRELATION_GROUPS,
    CROSS_GROUP_CORR,
    DEFAULT_PARAMS,
    INTRA_FINANCE_CORR,
    INTRA_TECH_CORR,
    SEED_PRICES,
    TICKER_PARAMS,
    TSLA_CORR,
)

logger = logging.getLogger(__name__)


class GBMSimulator:
    """Geometric Brownian Motion simulator for correlated stock prices.

    Pure math engine: holds current prices and advances them one step at a time.
    Does not interact with PriceCache or asyncio — that's SimulatorDataSource's job.
    """

    TRADING_SECONDS_PER_YEAR = 252 * 6.5 * 3600  # 5,896,800
    DEFAULT_DT = 0.5 / TRADING_SECONDS_PER_YEAR   # ~8.48e-8

    def __init__(
        self,
        tickers: list[str],
        dt: float = DEFAULT_DT,
        event_probability: float = 0.001,
    ) -> None:
        self._dt = dt
        self._event_prob = event_probability
        self._tickers: list[str] = []
        self._prices: dict[str, float] = {}
        self._params: dict[str, dict[str, float]] = {}
        self._cholesky: np.ndarray | None = None

        for ticker in tickers:
            self._add_ticker_internal(ticker)
        self._rebuild_cholesky()

    def step(self) -> dict[str, float]:
        """Advance all tickers by one time step. Returns {ticker: new_price}.

        This is the hot path — called every 500ms.
        """
        n = len(self._tickers)
        if n == 0:
            return {}

        # Generate n independent standard normal draws
        z_independent = np.random.standard_normal(n)

        # Apply Cholesky to get correlated draws
        if self._cholesky is not None:
            z_correlated = self._cholesky @ z_independent
        else:
            z_correlated = z_independent

        result: dict[str, float] = {}
        for i, ticker in enumerate(self._tickers):
            params = self._params[ticker]
            mu = params["mu"]
            sigma = params["sigma"]

            # GBM: S(t+dt) = S(t) * exp((mu - 0.5*sigma^2)*dt + sigma*sqrt(dt)*Z)
            drift = (mu - 0.5 * sigma ** 2) * self._dt
            diffusion = sigma * math.sqrt(self._dt) * z_correlated[i]
            self._prices[ticker] *= math.exp(drift + diffusion)

            # Random event: ~0.1% chance per tick per ticker
            if random.random() < self._event_prob:
                shock_magnitude = random.uniform(0.02, 0.05)
                shock_sign = random.choice([-1, 1])
                self._prices[ticker] *= 1 + shock_magnitude * shock_sign
                logger.debug(
                    "Random event on %s: %.1f%% %s",
                    ticker,
                    shock_magnitude * 100,
                    "up" if shock_sign > 0 else "down",
                )

            result[ticker] = round(self._prices[ticker], 2)

        return result

    def add_ticker(self, ticker: str) -> None:
        """Add a ticker to the simulation. Rebuilds the correlation matrix."""
        if ticker in self._prices:
            return
        self._add_ticker_internal(ticker)
        self._rebuild_cholesky()

    def remove_ticker(self, ticker: str) -> None:
        """Remove a ticker from the simulation. Rebuilds the correlation matrix."""
        if ticker not in self._prices:
            return
        self._tickers.remove(ticker)
        del self._prices[ticker]
        del self._params[ticker]
        self._rebuild_cholesky()

    def get_price(self, ticker: str) -> float | None:
        """Current price for a ticker, or None if not tracked."""
        return self._prices.get(ticker)

    def get_tickers(self) -> list[str]:
        """Return the current list of tracked tickers."""
        return list(self._tickers)

    # --- Internals ---

    def _add_ticker_internal(self, ticker: str) -> None:
        """Add a ticker without rebuilding Cholesky (for batch init)."""
        if ticker in self._prices:
            return
        self._tickers.append(ticker)
        self._prices[ticker] = SEED_PRICES.get(ticker, random.uniform(50.0, 300.0))
        self._params[ticker] = TICKER_PARAMS.get(ticker, dict(DEFAULT_PARAMS))

    def _rebuild_cholesky(self) -> None:
        """Rebuild the Cholesky decomposition of the ticker correlation matrix.

        Called whenever tickers are added or removed. O(n^2) but n < 50.
        """
        n = len(self._tickers)
        if n <= 1:
            self._cholesky = None
            return

        corr = np.eye(n)
        for i in range(n):
            for j in range(i + 1, n):
                rho = self._pairwise_correlation(self._tickers[i], self._tickers[j])
                corr[i, j] = rho
                corr[j, i] = rho

        self._cholesky = np.linalg.cholesky(corr)

    @staticmethod
    def _pairwise_correlation(t1: str, t2: str) -> float:
        """Determine correlation between two tickers based on sector grouping."""
        tech = CORRELATION_GROUPS["tech"]
        finance = CORRELATION_GROUPS["finance"]

        # TSLA is in the tech set but behaves independently
        if t1 == "TSLA" or t2 == "TSLA":
            return TSLA_CORR
        if t1 in tech and t2 in tech:
            return INTRA_TECH_CORR
        if t1 in finance and t2 in finance:
            return INTRA_FINANCE_CORR
        return CROSS_GROUP_CORR
```

### 7.2 SimulatorDataSource — Async Wrapper

Wraps `GBMSimulator` in a `MarketDataSource` that runs a background asyncio task.

```python
class SimulatorDataSource(MarketDataSource):
    """MarketDataSource backed by the GBM simulator.

    Runs a background asyncio task that calls GBMSimulator.step() every
    update_interval seconds and writes results to the PriceCache.
    """

    def __init__(
        self,
        price_cache: PriceCache,
        update_interval: float = 0.5,
        event_probability: float = 0.001,
    ) -> None:
        self._cache = price_cache
        self._interval = update_interval
        self._event_prob = event_probability
        self._sim: GBMSimulator | None = None
        self._task: asyncio.Task | None = None

    async def start(self, tickers: list[str]) -> None:
        self._sim = GBMSimulator(
            tickers=tickers,
            event_probability=self._event_prob,
        )
        # Seed the cache immediately so SSE has data on first tick
        for ticker in tickers:
            price = self._sim.get_price(ticker)
            if price is not None:
                self._cache.update(ticker=ticker, price=price)
        self._task = asyncio.create_task(self._run_loop(), name="simulator-loop")
        logger.info("Simulator started with %d tickers", len(tickers))

    async def stop(self) -> None:
        if self._task and not self._task.done():
            self._task.cancel()
            try:
                await self._task
            except asyncio.CancelledError:
                pass
        self._task = None
        logger.info("Simulator stopped")

    async def add_ticker(self, ticker: str) -> None:
        if self._sim:
            self._sim.add_ticker(ticker)
            price = self._sim.get_price(ticker)
            if price is not None:
                self._cache.update(ticker=ticker, price=price)
            logger.info("Simulator: added ticker %s", ticker)

    async def remove_ticker(self, ticker: str) -> None:
        if self._sim:
            self._sim.remove_ticker(ticker)
        self._cache.remove(ticker)
        logger.info("Simulator: removed ticker %s", ticker)

    def get_tickers(self) -> list[str]:
        return self._sim.get_tickers() if self._sim else []

    async def _run_loop(self) -> None:
        """Core loop: step the simulation, write to cache, sleep."""
        while True:
            try:
                if self._sim:
                    prices = self._sim.step()
                    for ticker, price in prices.items():
                        self._cache.update(ticker=ticker, price=price)
            except Exception:
                logger.exception("Simulator step failed")
            await asyncio.sleep(self._interval)
```

### Key Behaviors

| Behavior | Detail |
|----------|--------|
| **Immediate seeding** | `start()` populates the cache with seed prices *before* the loop begins, so SSE has data on the very first client connection. |
| **Graceful cancellation** | `stop()` cancels the task and awaits it, catching `CancelledError`. Clean shutdown during FastAPI lifespan teardown. |
| **Exception resilience** | The loop catches exceptions per-step, so a single bad tick doesn't kill the entire data feed. Logs at exception level for debugging. |
| **Dynamic tickers** | `add_ticker()` immediately seeds the cache so the new ticker has a price right away. `remove_ticker()` clears both the simulator state and the cache entry. |

---

## 8. Massive API Client — `massive_client.py`

Polls the Massive (formerly Polygon.io) REST API snapshot endpoint on a configurable interval. The synchronous Massive client runs in `asyncio.to_thread()` to avoid blocking the event loop.

### API Endpoint Used

```
GET /v2/snapshot/locale/us/markets/stocks/tickers?tickers=AAPL,GOOGL,MSFT
```

This endpoint returns current prices for **all requested tickers in a single API call** — critical for staying within the free tier's 5 requests/minute limit.

### Key Fields Extracted

From each ticker snapshot:
- `snap.last_trade.price` — current price for trading and display
- `snap.last_trade.timestamp` — Unix milliseconds (converted to seconds)

### Implementation

```python
from __future__ import annotations

import asyncio
import logging
from typing import Any

from .cache import PriceCache
from .interface import MarketDataSource

logger = logging.getLogger(__name__)


class MassiveDataSource(MarketDataSource):
    """MarketDataSource backed by the Massive (Polygon.io) REST API.

    Polls the snapshot endpoint for all watched tickers in a single API call,
    then writes results to the PriceCache.

    Rate limits:
      - Free tier: 5 req/min -> poll every 15s (default)
      - Paid tiers: higher limits -> poll every 2-5s
    """

    def __init__(
        self,
        api_key: str,
        price_cache: PriceCache,
        poll_interval: float = 15.0,
    ) -> None:
        self._api_key = api_key
        self._cache = price_cache
        self._interval = poll_interval
        self._tickers: list[str] = []
        self._task: asyncio.Task | None = None
        self._client: Any = None  # Lazy import

    async def start(self, tickers: list[str]) -> None:
        from massive import RESTClient

        self._client = RESTClient(api_key=self._api_key)
        self._tickers = list(tickers)

        # Immediate first poll so the cache has data right away
        await self._poll_once()

        self._task = asyncio.create_task(self._poll_loop(), name="massive-poller")
        logger.info(
            "Massive poller started: %d tickers, %.1fs interval",
            len(tickers), self._interval,
        )

    async def stop(self) -> None:
        if self._task and not self._task.done():
            self._task.cancel()
            try:
                await self._task
            except asyncio.CancelledError:
                pass
        self._task = None
        self._client = None
        logger.info("Massive poller stopped")

    async def add_ticker(self, ticker: str) -> None:
        ticker = ticker.upper().strip()
        if ticker not in self._tickers:
            self._tickers.append(ticker)
            logger.info("Massive: added ticker %s (will appear on next poll)", ticker)

    async def remove_ticker(self, ticker: str) -> None:
        ticker = ticker.upper().strip()
        self._tickers = [t for t in self._tickers if t != ticker]
        self._cache.remove(ticker)
        logger.info("Massive: removed ticker %s", ticker)

    def get_tickers(self) -> list[str]:
        return list(self._tickers)

    # --- Internal ---

    async def _poll_loop(self) -> None:
        """Poll on interval. First poll already happened in start()."""
        while True:
            await asyncio.sleep(self._interval)
            await self._poll_once()

    async def _poll_once(self) -> None:
        """Execute one poll cycle: fetch snapshots, update cache."""
        if not self._tickers or not self._client:
            return

        try:
            # Synchronous REST call — run in thread to avoid blocking event loop
            snapshots = await asyncio.to_thread(self._fetch_snapshots)
            processed = 0
            for snap in snapshots:
                try:
                    price = snap.last_trade.price
                    # Massive timestamps are Unix milliseconds -> seconds
                    timestamp = snap.last_trade.timestamp / 1000.0
                    self._cache.update(
                        ticker=snap.ticker,
                        price=price,
                        timestamp=timestamp,
                    )
                    processed += 1
                except (AttributeError, TypeError) as e:
                    logger.warning(
                        "Skipping snapshot for %s: %s",
                        getattr(snap, "ticker", "???"), e,
                    )
            logger.debug("Massive poll: updated %d/%d tickers", processed, len(self._tickers))

        except Exception as e:
            logger.error("Massive poll failed: %s", e)

    def _fetch_snapshots(self) -> list:
        """Synchronous call to the Massive REST API. Runs in a thread."""
        from massive.rest.models import SnapshotMarketType

        return self._client.get_snapshot_all(
            market_type=SnapshotMarketType.STOCKS,
            tickers=self._tickers,
        )
```

### Error Handling Philosophy

The Massive poller is intentionally resilient — it never crashes the application:

| Error | Behavior |
|-------|----------|
| **401 Unauthorized** | Logged. Poller keeps running (user might fix `.env` and restart). |
| **429 Rate Limited** | Logged. Next poll retries after `poll_interval` seconds. |
| **Network timeout** | Logged. Retries automatically on next cycle. |
| **Malformed snapshot** | Individual ticker skipped with warning. Other tickers still processed. |
| **All tickers fail** | Cache retains last-known prices. SSE keeps streaming stale data (better than nothing). |

### Lazy Import Strategy

`from massive import RESTClient` happens inside `start()`, not at module import time. This means:
- The `massive` package is only required when `MASSIVE_API_KEY` is set
- Students without a Massive API key don't need the package installed
- The simulator path has zero external dependencies beyond `numpy`

### Behavioral Difference from Simulator

| Aspect | Simulator | Massive |
|--------|-----------|---------|
| Update frequency | 500ms | 15s (free tier) |
| Data on `add_ticker()` | Immediate (seed price) | Next poll cycle (up to 15s delay) |
| `remove_ticker()` normalization | None (uppercase assumed) | `ticker.upper().strip()` |
| Offline behavior | Always works | Requires internet + valid API key |

---

## 9. Factory — `factory.py`

Selects the data source at startup based on the `MASSIVE_API_KEY` environment variable.

```python
from __future__ import annotations

import logging
import os

from .cache import PriceCache
from .interface import MarketDataSource

logger = logging.getLogger(__name__)


def create_market_data_source(price_cache: PriceCache) -> MarketDataSource:
    """Create the appropriate market data source based on environment variables.

    - MASSIVE_API_KEY set and non-empty -> MassiveDataSource (real market data)
    - Otherwise -> SimulatorDataSource (GBM simulation, default)

    Returns an unstarted source. Caller must await source.start(tickers).
    """
    api_key = os.environ.get("MASSIVE_API_KEY", "").strip()

    if api_key:
        from .massive_client import MassiveDataSource

        logger.info("Market data source: Massive API (real data)")
        return MassiveDataSource(api_key=api_key, price_cache=price_cache)
    else:
        from .simulator import SimulatorDataSource

        logger.info("Market data source: GBM Simulator")
        return SimulatorDataSource(price_cache=price_cache)
```

### Usage

```python
from app.market import PriceCache, create_market_data_source

cache = PriceCache()
source = create_market_data_source(cache)   # Reads env var
await source.start(["AAPL", "GOOGL", ...])  # Begin producing prices
```

### Edge Cases

- **Whitespace-only key**: `MASSIVE_API_KEY="   "` is treated as empty (stripped to `""`), so the simulator is used. This prevents a common configuration mistake.
- **Key set but invalid**: The Massive client will start, but the first poll will fail with 401. Prices will be empty until the user fixes their key.

---

## 10. SSE Streaming — `stream.py`

The SSE endpoint holds open a long-lived HTTP connection and pushes price updates to the client as `text/event-stream`.

```python
from __future__ import annotations

import asyncio
import json
import logging

from collections.abc import AsyncGenerator

from fastapi import APIRouter, Request
from fastapi.responses import StreamingResponse

from .cache import PriceCache

logger = logging.getLogger(__name__)


def create_stream_router(price_cache: PriceCache) -> APIRouter:
    """Create the SSE streaming router with a reference to the price cache.

    This factory pattern injects the PriceCache without globals.
    Creates a new APIRouter each call to avoid route double-registration.
    """
    router = APIRouter(prefix="/api/stream", tags=["streaming"])

    @router.get("/prices")
    async def stream_prices(request: Request) -> StreamingResponse:
        """SSE endpoint for live price updates.

        Streams all tracked ticker prices every ~500ms. The client connects
        with EventSource and receives events in the format:

            data: {"AAPL": {...}, "GOOGL": {...}, ...}

        Includes a retry directive so the browser auto-reconnects on
        disconnect (EventSource built-in behavior).
        """
        return StreamingResponse(
            _generate_events(price_cache, request),
            media_type="text/event-stream",
            headers={
                "Cache-Control": "no-cache",
                "Connection": "keep-alive",
                "X-Accel-Buffering": "no",  # Disable nginx buffering
            },
        )

    return router


async def _generate_events(
    price_cache: PriceCache,
    request: Request,
    interval: float = 0.5,
) -> AsyncGenerator[str, None]:
    """Async generator that yields SSE-formatted price events.

    Sends all prices every interval seconds, but only when the cache
    version has changed (avoiding redundant sends).
    """
    # Tell the client to retry after 1 second if the connection drops
    yield "retry: 1000\n\n"

    last_version = -1
    client_ip = request.client.host if request.client else "unknown"
    logger.info("SSE client connected: %s", client_ip)

    try:
        while True:
            if await request.is_disconnected():
                logger.info("SSE client disconnected: %s", client_ip)
                break

            current_version = price_cache.version
            if current_version != last_version:
                last_version = current_version
                prices = price_cache.get_all()

                if prices:
                    data = {
                        ticker: update.to_dict()
                        for ticker, update in prices.items()
                    }
                    payload = json.dumps(data)
                    yield f"data: {payload}\n\n"

            await asyncio.sleep(interval)
    except asyncio.CancelledError:
        logger.info("SSE stream cancelled for: %s", client_ip)
```

### SSE Wire Format

Each event the client receives:

```
data: {"AAPL":{"ticker":"AAPL","price":190.50,"previous_price":190.42,"timestamp":1707580800.5,"change":0.08,"change_percent":0.042,"direction":"up"},"GOOGL":{...}}

```

### Client-Side Consumption

```javascript
const eventSource = new EventSource('/api/stream/prices');

eventSource.onmessage = (event) => {
    const prices = JSON.parse(event.data);
    // prices = { "AAPL": { ticker, price, previous_price, change, direction, ... }, ... }
    Object.values(prices).forEach(update => {
        updateWatchlistRow(update.ticker, update);
        if (update.direction !== 'flat') {
            triggerPriceFlash(update.ticker, update.direction);
        }
    });
};

eventSource.onerror = () => {
    // EventSource auto-reconnects after the retry interval (1000ms)
    updateConnectionStatus('reconnecting');
};
```

### SSE Response Headers

| Header | Purpose |
|--------|---------|
| `Cache-Control: no-cache` | Prevents browser/proxy from caching the event stream |
| `Connection: keep-alive` | Keeps the TCP connection open for continuous streaming |
| `X-Accel-Buffering: no` | Disables nginx buffering if deployed behind nginx proxy |

### Why Poll-and-Push Instead of Event-Driven

The SSE endpoint polls the cache on a fixed 500ms interval rather than being notified when data changes. This is simpler and produces predictable, evenly-spaced updates. The frontend accumulates these into sparkline charts — regular spacing is important for clean visualization.

---

## 11. Module Exports — `__init__.py`

The `__init__.py` re-exports the public API so downstream code imports cleanly:

```python
"""Market data subsystem for AI Trading Workstation.

Public API:
    PriceUpdate              - Immutable price snapshot dataclass
    PriceCache               - Thread-safe in-memory price store
    MarketDataSource         - Abstract interface for data providers
    create_market_data_source - Factory: simulator or Massive API
    create_stream_router      - FastAPI SSE endpoint factory
"""

from .cache import PriceCache
from .factory import create_market_data_source
from .interface import MarketDataSource
from .models import PriceUpdate
from .stream import create_stream_router

__all__ = [
    "PriceUpdate",
    "PriceCache",
    "MarketDataSource",
    "create_market_data_source",
    "create_stream_router",
]
```

### Usage from Downstream Code

```python
# Clean imports — no reaching into submodules
from app.market import PriceCache, create_market_data_source, create_stream_router
```

---

## 12. FastAPI Lifecycle Integration

The market data system starts and stops with the FastAPI application using the `lifespan` context manager.

### Application Entry Point (`backend/app/main.py`)

```python
from contextlib import asynccontextmanager

from fastapi import FastAPI

from app.market import PriceCache, create_market_data_source, create_stream_router


@asynccontextmanager
async def lifespan(app: FastAPI):
    """Manage startup and shutdown of background services."""

    # --- STARTUP ---

    # 1. Create the shared price cache
    price_cache = PriceCache()
    app.state.price_cache = price_cache

    # 2. Create and start the market data source
    source = create_market_data_source(price_cache)
    app.state.market_source = source

    # 3. Load initial tickers from the database watchlist
    initial_tickers = await load_watchlist_tickers()  # reads from SQLite
    await source.start(initial_tickers)

    # 4. Register the SSE streaming router
    stream_router = create_stream_router(price_cache)
    app.include_router(stream_router)

    yield  # App is running

    # --- SHUTDOWN ---
    await source.stop()


app = FastAPI(title="AI Trading Workstation", lifespan=lifespan)
```

### Dependency Injection for Route Handlers

Other parts of the backend access the price cache and data source via FastAPI dependencies:

```python
from fastapi import Depends
from app.market import PriceCache, MarketDataSource


def get_price_cache() -> PriceCache:
    """FastAPI dependency: inject the shared price cache."""
    return app.state.price_cache


def get_market_source() -> MarketDataSource:
    """FastAPI dependency: inject the market data source."""
    return app.state.market_source
```

### Usage in Route Handlers

```python
from fastapi import APIRouter, Depends, HTTPException

router = APIRouter(prefix="/api")


@router.post("/portfolio/trade")
async def execute_trade(
    trade: TradeRequest,
    price_cache: PriceCache = Depends(get_price_cache),
):
    current_price = price_cache.get_price(trade.ticker)
    if current_price is None:
        raise HTTPException(404, f"No price available for {trade.ticker}")
    # ... execute trade at current_price ...


@router.get("/portfolio")
async def get_portfolio(
    price_cache: PriceCache = Depends(get_price_cache),
):
    all_prices = price_cache.get_all()
    # ... compute positions with live P&L using all_prices ...


@router.post("/watchlist")
async def add_to_watchlist(
    payload: WatchlistAdd,
    source: MarketDataSource = Depends(get_market_source),
):
    # Insert into database ...
    await source.add_ticker(payload.ticker)
    # ...


@router.delete("/watchlist/{ticker}")
async def remove_from_watchlist(
    ticker: str,
    source: MarketDataSource = Depends(get_market_source),
):
    # Remove from database ...
    await source.remove_ticker(ticker)
    # ...
```

### Startup Sequence

```
1. FastAPI app starts
2. lifespan() enters: creates PriceCache, creates data source
3. load_watchlist_tickers() reads from SQLite: ["AAPL", "GOOGL", ..., "NFLX"]
4. source.start(tickers) begins:
   - Simulator: creates GBMSimulator, seeds cache, starts async loop
   - Massive: creates RESTClient, does immediate first poll, starts poll loop
5. SSE router registered on the app
6. App is ready to serve requests
7. ... app runs ...
8. lifespan() exits: source.stop() cancels background task
```

---

## 13. Watchlist Coordination

When the watchlist changes (via REST API or LLM chat), the market data source must be notified to track the right set of tickers.

### Adding a Ticker

```
User (or LLM) -> POST /api/watchlist {ticker: "PYPL"}
  -> Insert into watchlist table (SQLite)
  -> await source.add_ticker("PYPL")
       Simulator: adds to GBMSimulator, rebuilds Cholesky, seeds cache
       Massive: appends to ticker list, appears on next poll
  -> Return success with current price (if available)
```

### Removing a Ticker

```
User (or LLM) -> DELETE /api/watchlist/PYPL
  -> Delete from watchlist table (SQLite)
  -> Check if user has an open position in PYPL
       If no position: await source.remove_ticker("PYPL")
       If position held: keep tracking (SSE needs prices for P&L)
  -> Return success
```

### Edge Case: Ticker Has an Open Position

Per PLAN.md section 6, the SSE stream must push prices for the **union of watchlist tickers AND position tickers**. If a user removes a ticker from the watchlist but still holds shares, the data source must keep tracking it for live P&L:

```python
@router.delete("/watchlist/{ticker}")
async def remove_from_watchlist(
    ticker: str,
    source: MarketDataSource = Depends(get_market_source),
):
    # Remove from watchlist table
    await db.delete_watchlist_entry(ticker)

    # Only stop tracking if no open position
    position = await db.get_position(ticker)
    if position is None or position.quantity == 0:
        await source.remove_ticker(ticker)

    return {"status": "ok"}
```

### Edge Case: Selling All Shares of an Unwatched Ticker

When a sell trade reduces a position to zero, and the ticker is not on the watchlist, the data source should stop tracking it:

```python
async def execute_sell(ticker: str, quantity: float, source: MarketDataSource):
    # ... execute the sell ...
    remaining = await db.get_position(ticker)
    if remaining is None or remaining.quantity == 0:
        is_watched = await db.is_in_watchlist(ticker)
        if not is_watched:
            await source.remove_ticker(ticker)
```

---

## 14. Data Flow Diagrams

### Price Update Flow (Simulator)

```
GBMSimulator.step()
    │
    │ returns {ticker: price}
    │
    ▼
SimulatorDataSource._run_loop()
    │
    │ for each ticker:
    │   cache.update(ticker, price)
    │
    ▼
PriceCache
    │
    │ _version incremented
    │ PriceUpdate created
    │
    ├──────────────────────────────────────┐
    │                                      │
    ▼                                      ▼
SSE _generate_events()              Trade Execution
    │                                      │
    │ if version changed:             price_cache.get_price("AAPL")
    │   yield data: {...}                  │
    │                                 use price for trade fill
    ▼
Browser EventSource
    │
    │ onmessage callback
    │
    ▼
React state update -> UI re-render
```

### Price Update Flow (Massive)

```
Massive REST API (Polygon.io)
    │
    │ GET /v2/snapshot/.../tickers?tickers=AAPL,GOOGL,...
    │
    ▼
MassiveDataSource._fetch_snapshots()     (runs in thread via asyncio.to_thread)
    │
    │ returns list of snapshot objects
    │
    ▼
MassiveDataSource._poll_once()
    │
    │ for each snapshot:
    │   cache.update(snap.ticker, snap.last_trade.price, snap.last_trade.timestamp/1000)
    │
    ▼
PriceCache                               (same downstream flow as simulator)
    │
    ├── SSE streaming
    ├── Portfolio valuation
    └── Trade execution
```

### Watchlist Change Flow

```
POST /api/watchlist {ticker: "PYPL"}
    │
    ├── db.insert_watchlist("PYPL")
    │
    └── await source.add_ticker("PYPL")
            │
            ├── Simulator path:
            │   GBMSimulator.add_ticker("PYPL")
            │     ├── seed price: random(50, 300)
            │     ├── params: DEFAULT_PARAMS
            │     └── _rebuild_cholesky()
            │   cache.update("PYPL", seed_price)
            │   -> PYPL has a price immediately
            │
            └── Massive path:
                _tickers.append("PYPL")
                -> PYPL appears on next poll (up to 15s delay)
```

---

## 15. Testing Strategy

### 15.1 Unit Tests — PriceUpdate Model

```python
class TestPriceUpdate:
    def test_direction_up(self):
        u = PriceUpdate(ticker="AAPL", price=191.0, previous_price=190.0)
        assert u.direction == "up"
        assert u.change == 1.0
        assert u.change_percent > 0

    def test_direction_down(self):
        u = PriceUpdate(ticker="AAPL", price=189.0, previous_price=190.0)
        assert u.direction == "down"

    def test_direction_flat(self):
        u = PriceUpdate(ticker="AAPL", price=190.0, previous_price=190.0)
        assert u.direction == "flat"
        assert u.change == 0.0

    def test_frozen(self):
        u = PriceUpdate(ticker="AAPL", price=190.0, previous_price=190.0)
        with pytest.raises(FrozenInstanceError):
            u.price = 200.0

    def test_to_dict_contains_all_fields(self):
        u = PriceUpdate(ticker="AAPL", price=191.0, previous_price=190.0, timestamp=1000.0)
        d = u.to_dict()
        assert set(d.keys()) == {
            "ticker", "price", "previous_price", "timestamp",
            "change", "change_percent", "direction",
        }

    def test_zero_previous_price(self):
        u = PriceUpdate(ticker="X", price=10.0, previous_price=0.0)
        assert u.change_percent == 0.0  # No division by zero
```

### 15.2 Unit Tests — PriceCache

```python
class TestPriceCache:
    def test_update_and_get(self):
        cache = PriceCache()
        update = cache.update("AAPL", 190.50)
        assert cache.get("AAPL") == update
        assert update.price == 190.50

    def test_first_update_is_flat(self):
        cache = PriceCache()
        update = cache.update("AAPL", 190.50)
        assert update.direction == "flat"
        assert update.previous_price == 190.50

    def test_version_increments(self):
        cache = PriceCache()
        v0 = cache.version
        cache.update("AAPL", 190.00)
        assert cache.version == v0 + 1

    def test_remove(self):
        cache = PriceCache()
        cache.update("AAPL", 190.00)
        cache.remove("AAPL")
        assert cache.get("AAPL") is None

    def test_get_all_returns_copy(self):
        cache = PriceCache()
        cache.update("AAPL", 190.0)
        all_prices = cache.get_all()
        all_prices["FAKE"] = None  # Mutate the copy
        assert "FAKE" not in cache  # Original unaffected

    def test_prices_rounded_to_cents(self):
        cache = PriceCache()
        cache.update("AAPL", 190.456789)
        assert cache.get_price("AAPL") == 190.46
```

### 15.3 Unit Tests — GBMSimulator

```python
class TestGBMSimulator:
    def test_step_returns_all_tickers(self):
        sim = GBMSimulator(tickers=["AAPL", "GOOGL"])
        result = sim.step()
        assert set(result.keys()) == {"AAPL", "GOOGL"}

    def test_prices_are_positive(self):
        """GBM prices can never go negative (exp() is always positive)."""
        sim = GBMSimulator(tickers=["AAPL"])
        for _ in range(10_000):
            prices = sim.step()
            assert prices["AAPL"] > 0

    def test_initial_prices_match_seeds(self):
        sim = GBMSimulator(tickers=["AAPL"])
        assert sim.get_price("AAPL") == SEED_PRICES["AAPL"]

    def test_add_and_remove_ticker(self):
        sim = GBMSimulator(tickers=["AAPL"])
        sim.add_ticker("TSLA")
        assert "TSLA" in sim.step()
        sim.remove_ticker("TSLA")
        assert "TSLA" not in sim.step()

    def test_cholesky_rebuilds_on_add(self):
        sim = GBMSimulator(tickers=["AAPL"])
        assert sim._cholesky is None  # Only 1 ticker
        sim.add_ticker("GOOGL")
        assert sim._cholesky is not None  # Now 2 tickers

    def test_unknown_ticker_gets_random_seed(self):
        sim = GBMSimulator(tickers=["ZZZZ"])
        price = sim.get_price("ZZZZ")
        assert 50.0 <= price <= 300.0

    def test_full_10_ticker_cholesky(self):
        """Verify Cholesky decomposition succeeds for all 10 default tickers."""
        tickers = list(SEED_PRICES.keys())
        sim = GBMSimulator(tickers=tickers)
        assert sim._cholesky is not None
        assert sim._cholesky.shape == (10, 10)
```

### 15.4 Integration Tests — SimulatorDataSource

```python
@pytest.mark.asyncio
class TestSimulatorDataSource:
    async def test_start_populates_cache(self):
        cache = PriceCache()
        source = SimulatorDataSource(price_cache=cache, update_interval=0.1)
        await source.start(["AAPL", "GOOGL"])
        assert cache.get("AAPL") is not None
        assert cache.get("GOOGL") is not None
        await source.stop()

    async def test_prices_update_over_time(self):
        cache = PriceCache()
        source = SimulatorDataSource(price_cache=cache, update_interval=0.05)
        await source.start(["AAPL"])
        v0 = cache.version
        await asyncio.sleep(0.3)
        assert cache.version > v0  # Cache was updated
        await source.stop()

    async def test_stop_is_idempotent(self):
        cache = PriceCache()
        source = SimulatorDataSource(price_cache=cache, update_interval=0.1)
        await source.start(["AAPL"])
        await source.stop()
        await source.stop()  # Should not raise

    async def test_add_ticker_seeds_cache(self):
        cache = PriceCache()
        source = SimulatorDataSource(price_cache=cache, update_interval=0.1)
        await source.start(["AAPL"])
        await source.add_ticker("TSLA")
        assert cache.get("TSLA") is not None
        await source.stop()
```

### 15.5 Unit Tests — MassiveDataSource (Mocked)

```python
def _make_snapshot(ticker: str, price: float, timestamp_ms: int) -> MagicMock:
    """Create a mock Massive snapshot object."""
    snap = MagicMock()
    snap.ticker = ticker
    snap.last_trade.price = price
    snap.last_trade.timestamp = timestamp_ms
    return snap


@pytest.mark.asyncio
class TestMassiveDataSource:
    async def test_poll_updates_cache(self):
        cache = PriceCache()
        source = MassiveDataSource(api_key="test", price_cache=cache, poll_interval=60.0)
        source._tickers = ["AAPL"]

        mock_snaps = [_make_snapshot("AAPL", 190.50, 1707580800000)]
        with patch.object(source, "_fetch_snapshots", return_value=mock_snaps):
            await source._poll_once()

        assert cache.get_price("AAPL") == 190.50

    async def test_malformed_snapshot_skipped(self):
        cache = PriceCache()
        source = MassiveDataSource(api_key="test", price_cache=cache, poll_interval=60.0)
        source._tickers = ["AAPL", "BAD"]

        good = _make_snapshot("AAPL", 190.50, 1707580800000)
        bad = MagicMock()
        bad.ticker = "BAD"
        bad.last_trade = None  # Will cause AttributeError

        with patch.object(source, "_fetch_snapshots", return_value=[good, bad]):
            await source._poll_once()

        assert cache.get_price("AAPL") == 190.50
        assert cache.get_price("BAD") is None

    async def test_timestamp_conversion(self):
        cache = PriceCache()
        source = MassiveDataSource(api_key="test", price_cache=cache, poll_interval=60.0)
        source._tickers = ["AAPL"]

        mock_snaps = [_make_snapshot("AAPL", 190.50, 1707580800000)]
        with patch.object(source, "_fetch_snapshots", return_value=mock_snaps):
            await source._poll_once()

        update = cache.get("AAPL")
        assert update.timestamp == 1707580800.0  # ms -> seconds
```

### 15.6 Unit Tests — Factory

```python
class TestFactory:
    def test_no_key_returns_simulator(self, monkeypatch):
        monkeypatch.delenv("MASSIVE_API_KEY", raising=False)
        cache = PriceCache()
        source = create_market_data_source(cache)
        assert isinstance(source, SimulatorDataSource)

    def test_empty_key_returns_simulator(self, monkeypatch):
        monkeypatch.setenv("MASSIVE_API_KEY", "")
        cache = PriceCache()
        source = create_market_data_source(cache)
        assert isinstance(source, SimulatorDataSource)

    def test_whitespace_key_returns_simulator(self, monkeypatch):
        monkeypatch.setenv("MASSIVE_API_KEY", "   ")
        cache = PriceCache()
        source = create_market_data_source(cache)
        assert isinstance(source, SimulatorDataSource)

    def test_valid_key_returns_massive(self, monkeypatch):
        monkeypatch.setenv("MASSIVE_API_KEY", "real-key-here")
        cache = PriceCache()
        source = create_market_data_source(cache)
        assert isinstance(source, MassiveDataSource)
```

### Test Coverage Targets

| Module | Target | Notes |
|--------|--------|-------|
| models.py | 100% | All branches reachable |
| cache.py | 100% | All methods exercised |
| simulator.py | >95% | Exception handler in `_run_loop` hard to trigger |
| massive_client.py | ~55% | API methods require real/mocked `massive` package |
| factory.py | 100% | All env var branches tested |
| interface.py | 100% | Abstract methods covered by implementations |
| stream.py | ~30% | SSE generator requires running ASGI server |

---

## 16. Error Handling & Edge Cases

### 16.1 Startup Errors

| Scenario | Behavior |
|----------|----------|
| `massive` package not installed but `MASSIVE_API_KEY` set | `ImportError` on `start()`. Application fails to start. Fix: install `massive` or unset the env var. |
| Invalid `MASSIVE_API_KEY` | Application starts. First poll logs 401 error. Cache remains empty. SSE sends no data until key is fixed. |
| Network unreachable at startup (Massive) | `_poll_once()` in `start()` logs error. Loop begins, retries on each interval. |

### 16.2 Runtime Errors

| Scenario | Behavior |
|----------|----------|
| Simulator `step()` raises exception | Caught in `_run_loop()`, logged. Next tick retries. |
| Massive API returns 429 (rate limited) | Caught in `_poll_once()`, logged. Next poll after interval. |
| Massive API returns malformed data | Per-snapshot try/catch. Bad ticker skipped, others processed. |
| SSE client disconnects | `request.is_disconnected()` returns True. Generator stops cleanly. |
| All SSE clients disconnect | No effect on data source. It keeps running and updating the cache. |

### 16.3 Edge Cases

| Scenario | Behavior |
|----------|----------|
| `add_ticker()` with already-tracked ticker | No-op. No duplicate entries. |
| `remove_ticker()` with non-tracked ticker | No-op. No error. |
| `stop()` called twice | Safe. Second call is a no-op (task already None or done). |
| Empty ticker list on `start()` | Simulator creates with 0 tickers. `step()` returns `{}`. No SSE data sent. |
| Ticker added to Massive source | Appears on next poll (up to 15s delay). No immediate data. |
| Ticker added to Simulator source | Immediate seed price in cache. Data available instantly. |
| `PriceCache.get()` for unknown ticker | Returns `None`. API endpoints return `null` price fields. |

---

## 17. Configuration Reference

### Environment Variables

| Variable | Required | Default | Description |
|----------|----------|---------|-------------|
| `MASSIVE_API_KEY` | No | (empty) | Massive/Polygon.io API key. If set, uses real market data. If empty, uses simulator. |

### Tunable Constants (In Code)

| Constant | Location | Default | Description |
|----------|----------|---------|-------------|
| `update_interval` | `SimulatorDataSource.__init__` | `0.5` (seconds) | How often the simulator steps |
| `event_probability` | `GBMSimulator.__init__` | `0.001` (0.1%) | Per-tick probability of a random shock event |
| `poll_interval` | `MassiveDataSource.__init__` | `15.0` (seconds) | How often to poll the Massive API |
| `DEFAULT_DT` | `GBMSimulator` class | `~8.48e-8` | GBM time step (500ms as fraction of trading year) |
| SSE interval | `_generate_events` | `0.5` (seconds) | How often SSE checks for new data |
| SSE retry | `_generate_events` | `1000` (ms) | Browser reconnection delay |

### Dependencies

| Package | Version | Used By | Required When |
|---------|---------|---------|---------------|
| `numpy` | >=2.0.0 | GBMSimulator (Cholesky, random normals) | Always |
| `fastapi` | >=0.115.0 | SSE streaming router | Always |
| `uvicorn` | >=0.32.0 | Application server | Always |
| `massive` | >=1.0.0 | MassiveDataSource | Only when `MASSIVE_API_KEY` is set |

---

## Appendix: Quick Reference

### Importing the Market Data API

```python
from app.market import PriceCache, create_market_data_source, create_stream_router
```

### Full Lifecycle Example

```python
# Startup
cache = PriceCache()
source = create_market_data_source(cache)  # Reads MASSIVE_API_KEY
await source.start(["AAPL", "GOOGL", "MSFT", "AMZN", "TSLA",
                     "NVDA", "META", "JPM", "V", "NFLX"])

# Read prices (from any route handler)
update = cache.get("AAPL")          # PriceUpdate or None
price = cache.get_price("AAPL")     # float or None
all_prices = cache.get_all()        # dict[str, PriceUpdate]

# Dynamic watchlist
await source.add_ticker("PYPL")
await source.remove_ticker("GOOGL")

# Shutdown
await source.stop()
```
