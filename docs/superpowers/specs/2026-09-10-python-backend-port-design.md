# Python Backend Port Design

## Goal

Replace the TypeScript/Hono backend with a Python/FastAPI backend while keeping the existing React frontend and API contract working.

## Scope

Port the backend runtime only. Keep `frontend/` as React/Vite. Preserve port `5611`, the `/api/*` route shapes, `/ws`, SQLite persistence, and the current paper-trading behavior.

Do not add new trading features, symbols, exchanges, auth models, or UI flows during the port.

## Architecture

Use FastAPI for HTTP and WebSocket handling, `sqlite3` from the Python standard library for persistence, `ccxt` for Hyperliquid market data, `apscheduler` for scheduled jobs, and `httpx` for outbound AI/news calls.

Create `backend_py/` alongside the existing `backend/` during the migration. Once it passes route and service checks, point root scripts and Docker at `backend_py`. Keep the TypeScript backend in the repo until the Python backend is verified, then remove or archive it in a final cleanup.

## Runtime Contract

- `PORT` defaults to `5611`.
- `DATABASE_PATH` defaults to a backend-local `data.db`.
- API routes remain:
  - `GET /api/health`
  - `/api/account`
  - `/api/config`
  - `/api/crypto`
  - `/api/market`
  - `/api/orders`
  - `/api/ranking`
  - `GET /ws`
- Static frontend build is served from the backend runtime image.
- CORS remains permissive to match current behavior.

## Data Contract

Use the existing SQLite schema exactly. `ensure_schema()` emits the same table and index DDL as `backend/src/db/client.ts`. `seed_database()` preserves the current default user/account behavior.

Rows should serialize with the same snake_case JSON fields the frontend receives today.

## Trading Contract

Port these service responsibilities:

- Market data: Hyperliquid symbol formatting, price fetches, klines, market status, symbol list.
- Price cache: in-memory last-price cache with current behavior.
- Orders: create orders, execute market/limit orders, update accounts/positions/trades.
- Leverage: taker fee, margin, interest, long/short position accounting.
- AI decisions: build prompt, call OpenAI-compatible chat completions endpoint, parse JSON, validate symbol/direction/portion/leverage.
- Scheduler: initialize recurring market data, order matching, random/AI trading jobs.
- WebSocket: accept client messages and push account/order/trade/position snapshots.

## Testing

Use pytest. Start with the smallest checks that guard the risky behavior:

- Schema creation creates expected tables.
- Seed creates the default user/account once.
- Symbol formatting maps mainstream bare symbols to Hyperliquid perps.
- Decision parsing accepts fenced JSON and cleaned model output.
- Order execution updates cash, positions, orders, and trades for a simple market buy.
- FastAPI smoke tests cover health and one account/crypto route.

Network-dependent CCXT and AI calls stay behind small functions that can be monkeypatched in tests.

## Migration Sequence

1. Add Python package, app entrypoint, database bootstrap, and seed.
2. Add repositories and serializers for current DB rows.
3. Port market data, price cache, and Hyperliquid helpers.
4. Port order/account/config routes and order execution.
5. Port AI decision and scheduler.
6. Port WebSocket snapshot flow.
7. Update root scripts and Docker to run the Python backend.
8. Run tests and a local server smoke check.

## Deliberate Simplifications

Keep raw `sqlite3` instead of adding SQLAlchemy. The schema is small, the current backend already owns explicit DDL, and avoiding an ORM reduces porting surface. If query complexity grows, add SQLAlchemy later with model tests around every migrated query.

Keep `backend_py/` beside `backend/` during migration instead of editing in place. This leaves the current backend available for comparison until the Python one is runnable.
