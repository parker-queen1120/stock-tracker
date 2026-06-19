# Architecture

## Overview
A price-tracking system with a Rust + Tauri desktop client, a Python + FastAPI
backend, and an async scraping/ingestion service. The client never touches the
database directly — all access goes through the API.

## System Diagram

```mermaid
graph TD
    subgraph Client["Desktop Client (Rust + Tauri)"]
        UI[UI: Watchlists, Price Charts, Alerts]
    end

    subgraph Backend["Backend (Python + FastAPI)"]
        API[REST API Layer]
        Scraper[Async Ingestion Service<br/>scheduled jobs]
    end

    subgraph External["External Data Sources"]
        YF[yfinance]
        AV[Alpha Vantage]
    end

    DB[(PostgreSQL)]

    UI -->|HTTP/JSON requests| API
    API -->|reads/writes| DB
    Scraper -->|fetches prices| YF
    Scraper -->|fetches prices| AV
    Scraper -->|writes snapshots| DB
```

## Request Flow Example: Adding a stock and viewing history

```mermaid
sequenceDiagram
    participant User
    participant Client as Desktop Client
    participant API as FastAPI Backend
    participant DB as PostgreSQL
    participant Scraper as Ingestion Service
    participant Source as yfinance/Alpha Vantage

    User->>Client: Add "AAPL" to watchlist
    Client->>API: POST /watchlists/{id}/items
    API->>DB: Insert item + watchlist_items row
    API-->>Client: 201 Created

    loop Scheduled interval
        Scraper->>DB: Get items needing price update
        Scraper->>Source: Fetch current price
        Source-->>Scraper: Price data
        Scraper->>DB: Insert price_snapshot
    end

    User->>Client: View price history for "AAPL"
    Client->>API: GET /items/{id}/price-history
    API->>DB: Query price_snapshots
    DB-->>API: Snapshot rows
    API-->>Client: JSON price history
    Client->>User: Render chart
```

## Database Schema

```sql
CREATE TABLE items (
    id INTEGER PRIMARY KEY,
    category TEXT NOT NULL,        -- 'stock', 'mtg_card', etc.
    name TEXT NOT NULL,
    external_id TEXT,              -- ticker symbol, Scryfall ID, etc.
    metadata JSON,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE sources (
    id INTEGER PRIMARY KEY,
    name TEXT NOT NULL,            -- 'yfinance', 'Alpha Vantage', 'TCGPlayer'
    base_url TEXT,
    scrape_config JSON              -- {"method": "api"|"scrape", ...}
);

CREATE TABLE price_snapshots (
    id INTEGER PRIMARY KEY,
    item_id INTEGER REFERENCES items(id),
    source_id INTEGER REFERENCES sources(id),
    price DECIMAL(10,2) NOT NULL,
    currency TEXT DEFAULT 'USD',
    condition TEXT,                 -- card condition; null for stocks
    captured_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE users (
    id INTEGER PRIMARY KEY,
    email TEXT UNIQUE NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE watchlists (
    id INTEGER PRIMARY KEY,
    user_id INTEGER REFERENCES users(id),
    name TEXT NOT NULL
);

CREATE TABLE watchlist_items (
    watchlist_id INTEGER REFERENCES watchlists(id),
    item_id INTEGER REFERENCES items(id),
    alert_threshold DECIMAL(10,2),
    PRIMARY KEY (watchlist_id, item_id)
);
```

## Design Decisions

- **`metadata JSON` on items**: avoids rigid per-category columns. Tradeoff:
  harder to index/query inside JSON on some DBs (Postgres JSONB handles this
  reasonably well).
- **Append-only `price_snapshots`**: never update a price, always insert a new
  row. This is what makes the system a tracker, not a lookup tool. Indexed on
  `(item_id, captured_at)` for the main query pattern.
- **`scrape_config` on sources**: keeps ingestion logic data-driven (adapter
  pattern) instead of hardcoded per-site branches. Adding a new source is a
  config entry + small adapter, not new core logic.
- **Client never touches the DB directly**: all access via the API. Keeps the
  desktop client, and any future client (mobile, web), interchangeable.

## Tech Stack

| Layer | Technology |
|---|---|
| Desktop client | Rust + Tauri |
| Backend API | Python + FastAPI |
| Ingestion service | Python (async, `aiohttp`) |
| Database | PostgreSQL |
| Data sources | yfinance, Alpha Vantage |