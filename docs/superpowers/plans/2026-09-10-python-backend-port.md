# Python Backend Port Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Replace the TypeScript/Hono backend with a Python/FastAPI backend without changing the React frontend contract.

**Architecture:** Build a `backend_py/` FastAPI app beside the current backend, preserve the same SQLite schema and `/api/*` routes, then switch scripts and Docker after verification. Use small modules that mirror the existing backend responsibilities so behavior can be ported and tested slice by slice.

**Tech Stack:** Python 3.11+, FastAPI, Uvicorn, pytest, httpx, ccxt, apscheduler, sqlite3.

**Spec:** `docs/superpowers/specs/2026-09-10-python-backend-port-design.md`

## Global Constraints

- Keep `frontend/` as React/Vite.
- Preserve port `5611`, `/api/*`, `/ws`, SQLite persistence, and current paper-trading behavior.
- Do not add new trading features, symbols, exchanges, auth models, or UI flows during the port.
- Use the existing SQLite schema exactly.
- Rows serialize with the same snake_case JSON fields the frontend receives today.
- Network-dependent CCXT and AI calls stay behind small functions that can be monkeypatched in tests.

---

## File Structure

- `backend_py/pyproject.toml`: Python dependencies and pytest config.
- `backend_py/app/main.py`: FastAPI app, route mounting, static serving, startup/shutdown.
- `backend_py/app/db.py`: SQLite connection, schema DDL, transaction helper.
- `backend_py/app/seed.py`: default trading config/user/account seed.
- `backend_py/app/models.py`: `TypedDict` row contracts and constants.
- `backend_py/app/serializers.py`: DB row to frontend JSON shapes.
- `backend_py/app/repos.py`: small SQLite query helpers.
- `backend_py/app/services/market_data.py`: price cache and Hyperliquid calls.
- `backend_py/app/services/orders.py`: order creation/execution/accounting.
- `backend_py/app/services/ai.py`: AI prompt, decision parsing, validation helpers.
- `backend_py/app/services/scheduler.py`: APScheduler jobs and lifecycle.
- `backend_py/app/api/*.py`: FastAPI routers matching current route groups.
- `backend_py/app/ws.py`: WebSocket snapshot flow.
- `backend_py/tests/*.py`: focused pytest checks.
- `package.json`: root scripts switch backend commands to Python.
- `Dockerfile`: build React with Node, run Python backend with built static files.

### Task 1: Python App Skeleton And Schema

**Files:**
- Create: `backend_py/pyproject.toml`
- Create: `backend_py/app/__init__.py`
- Create: `backend_py/app/db.py`
- Create: `backend_py/app/models.py`
- Create: `backend_py/app/seed.py`
- Create: `backend_py/app/main.py`
- Create: `backend_py/tests/test_db_seed.py`

**Interfaces:**
- Produces: `get_db() -> sqlite3.Connection`, `ensure_schema(conn: sqlite3.Connection) -> None`, `seed_database(conn: sqlite3.Connection) -> None`, `create_app() -> FastAPI`.

- [ ] **Step 1: Write the failing schema/seed test**

```python
from backend_py.app.db import ensure_schema, get_db
from backend_py.app.seed import seed_database


def test_schema_and_seed_create_default_account(tmp_path, monkeypatch):
    db_path = tmp_path / "data.db"
    monkeypatch.setenv("DATABASE_PATH", str(db_path))
    conn = get_db()
    ensure_schema(conn)
    seed_database(conn)
    seed_database(conn)

    tables = {
        row[0]
        for row in conn.execute(
            "select name from sqlite_master where type = 'table'"
        ).fetchall()
    }
    assert {"users", "accounts", "orders", "positions", "trades"}.issubset(tables)
    assert conn.execute("select count(*) from users").fetchone()[0] == 1
    assert conn.execute("select name, account_type from accounts").fetchone() == (
        "GPT",
        "AI",
    )
```

- [ ] **Step 2: Run test to verify it fails**

Run: `cd backend_py && pytest tests/test_db_seed.py -v`
Expected: FAIL with `ModuleNotFoundError` for `backend_py`.

- [ ] **Step 3: Add minimal implementation**

Create the package, paste the DDL from `backend/src/db/client.ts` into `ensure_schema()`, and port the seed logic from `backend/src/db/seed.ts`.

- [ ] **Step 4: Run test to verify it passes**

Run: `cd backend_py && pytest tests/test_db_seed.py -v`
Expected: PASS.

### Task 2: Market Data Service

**Files:**
- Create: `backend_py/app/services/__init__.py`
- Create: `backend_py/app/services/market_data.py`
- Create: `backend_py/tests/test_market_data.py`

**Interfaces:**
- Consumes: `get_db()`
- Produces: `format_symbol(symbol: str) -> str`, `get_last_price(symbol: str, market: str = "CRYPTO") -> float | None`, `get_kline_data(symbol: str, period: str = "1d", count: int = 100) -> list[dict]`, `get_market_status(symbol: str, market: str = "CRYPTO") -> dict`, `get_all_symbols() -> list[str]`.

- [ ] **Step 1: Write the failing symbol-format test**

```python
from backend_py.app.services.market_data import format_symbol


def test_format_symbol_uses_perps_for_mainstream_symbols():
    assert format_symbol("BTC") == "BTC/USDC:USDC"
    assert format_symbol("eth") == "ETH/USDC:USDC"
    assert format_symbol("PURR") == "PURR/USDC"
    assert format_symbol("BTC/USDC") == "BTC/USDC:USDC"
```

- [ ] **Step 2: Run test to verify it fails**

Run: `cd backend_py && pytest tests/test_market_data.py -v`
Expected: FAIL because `market_data.py` does not exist.

- [ ] **Step 3: Port minimal market data helpers**

Implement `MAINSTREAM_CRYPTOS`, `TIMEFRAME_MAP`, `format_symbol()`, a module-level `ccxt.hyperliquid` client, and simple in-memory cache `{(symbol, market): (price, timestamp)}`.

- [ ] **Step 4: Run test to verify it passes**

Run: `cd backend_py && pytest tests/test_market_data.py -v`
Expected: PASS.

### Task 3: Core Repositories And Serializers

**Files:**
- Create: `backend_py/app/repos.py`
- Create: `backend_py/app/serializers.py`
- Create: `backend_py/tests/test_serializers.py`

**Interfaces:**
- Consumes: `sqlite3.Connection`
- Produces: `row_to_dict(row: sqlite3.Row) -> dict`, `serialize_account(row) -> dict`, `get_accounts(conn) -> list[dict]`, `get_default_account(conn) -> dict | None`.

- [ ] **Step 1: Write the failing serializer test**

```python
from backend_py.app.db import ensure_schema, get_db
from backend_py.app.repos import get_accounts
from backend_py.app.seed import seed_database


def test_get_accounts_returns_frontend_fields(tmp_path, monkeypatch):
    monkeypatch.setenv("DATABASE_PATH", str(tmp_path / "data.db"))
    conn = get_db()
    ensure_schema(conn)
    seed_database(conn)

    account = get_accounts(conn)[0]

    assert account["name"] == "GPT"
    assert account["current_cash"] == 10000
    assert "currentCash" not in account
```

- [ ] **Step 2: Run test to verify it fails**

Run: `cd backend_py && pytest tests/test_serializers.py -v`
Expected: FAIL because `repos.py` does not exist.

- [ ] **Step 3: Add repository and serializer helpers**

Use `sqlite3.Row` and return snake_case dictionaries directly.

- [ ] **Step 4: Run test to verify it passes**

Run: `cd backend_py && pytest tests/test_serializers.py -v`
Expected: PASS.

### Task 4: Account, Config, Crypto, And Market Routes

**Files:**
- Create: `backend_py/app/api/__init__.py`
- Create: `backend_py/app/api/account.py`
- Create: `backend_py/app/api/config.py`
- Create: `backend_py/app/api/crypto.py`
- Create: `backend_py/app/api/market.py`
- Modify: `backend_py/app/main.py`
- Create: `backend_py/tests/test_api_smoke.py`

**Interfaces:**
- Consumes: `create_app()`, repositories, market data service.
- Produces: FastAPI routers mounted at `/api/account`, `/api/config`, `/api/crypto`, `/api/market`, plus `GET /api/health`.

- [ ] **Step 1: Write failing API smoke tests**

```python
from fastapi.testclient import TestClient

from backend_py.app.main import create_app


def test_health_route():
    client = TestClient(create_app())
    assert client.get("/api/health").json()["status"] == "healthy"


def test_crypto_symbols_route(monkeypatch):
    monkeypatch.setattr(
        "backend_py.app.api.crypto.get_all_symbols",
        lambda: ["BTC/USDC:USDC"],
    )
    client = TestClient(create_app())
    assert client.get("/api/crypto/symbols").json() == ["BTC/USDC:USDC"]
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `cd backend_py && pytest tests/test_api_smoke.py -v`
Expected: FAIL because routes are not mounted.

- [ ] **Step 3: Implement minimal routes**

Port read routes first. Return the same JSON field names as current Hono routes.

- [ ] **Step 4: Run tests to verify they pass**

Run: `cd backend_py && pytest tests/test_api_smoke.py -v`
Expected: PASS.

### Task 5: Order Execution

**Files:**
- Create: `backend_py/app/services/orders.py`
- Create: `backend_py/app/api/orders.py`
- Modify: `backend_py/app/main.py`
- Create: `backend_py/tests/test_orders.py`

**Interfaces:**
- Consumes: `get_last_price(symbol, market)`, repositories.
- Produces: `create_order(conn, account_id: int, symbol: str, name: str, side: str, order_type: str, price: float | None, quantity: float, leverage: int = 1) -> dict`, `execute_order(conn, order: dict) -> bool`.

- [ ] **Step 1: Write failing market-buy accounting test**

```python
from backend_py.app.db import ensure_schema, get_db
from backend_py.app.seed import seed_database
from backend_py.app.services.orders import create_order, execute_order


def test_market_buy_creates_position_order_and_trade(tmp_path, monkeypatch):
    monkeypatch.setenv("DATABASE_PATH", str(tmp_path / "data.db"))
    monkeypatch.setattr("backend_py.app.services.orders.get_last_price", lambda *_: 100.0)
    conn = get_db()
    ensure_schema(conn)
    seed_database(conn)

    order = create_order(conn, 1, "BTC", "Bitcoin", "BUY", "MARKET", None, 2, 1)
    assert execute_order(conn, order) is True

    account_cash = conn.execute("select current_cash from accounts where id = 1").fetchone()[0]
    position = conn.execute("select quantity, avg_cost from positions where symbol = 'BTC'").fetchone()
    trade_count = conn.execute("select count(*) from trades").fetchone()[0]
    assert account_cash == 9799.8
    assert position == (2, 100)
    assert trade_count == 1
```

- [ ] **Step 2: Run test to verify it fails**

Run: `cd backend_py && pytest tests/test_orders.py -v`
Expected: FAIL because `services.orders` does not exist.

- [ ] **Step 3: Port minimal spot market order logic**

Implement market buy/sell parity with current constants: commission rate `0.001`, minimum commission `0.1`, order status `FILLED`, filled quantity equals quantity.

- [ ] **Step 4: Run test to verify it passes**

Run: `cd backend_py && pytest tests/test_orders.py -v`
Expected: PASS.

### Task 6: AI Decision Logic

**Files:**
- Create: `backend_py/app/services/ai.py`
- Create: `backend_py/tests/test_ai.py`

**Interfaces:**
- Consumes: account dictionaries, portfolio dictionaries, price dictionaries.
- Produces: `SUPPORTED_SYMBOLS`, `build_prompt(portfolio, prices, news_section) -> str`, `parse_decision_text(text: str) -> dict`, `is_default_api_key(api_key: str | None) -> bool`.

- [ ] **Step 1: Write failing AI parsing test**

```python
from backend_py.app.services.ai import parse_decision_text


def test_parse_decision_text_accepts_json_fence_and_smart_quotes():
    decision = parse_decision_text(
        "```json\n{“operation”: “open”, “symbol”: “BTC”, "
        "“direction”: “long”, “target_portion_of_balance”: 0.2, "
        "“leverage”: 3, “reason”: “ok”}\n```"
    )
    assert decision["operation"] == "open"
    assert decision["symbol"] == "BTC"
    assert decision["leverage"] == 3
```

- [ ] **Step 2: Run test to verify it fails**

Run: `cd backend_py && pytest tests/test_ai.py -v`
Expected: FAIL because `services.ai` does not exist.

- [ ] **Step 3: Port parsing and prompt helpers**

Use the same supported symbols and prompt constraints as `backend/src/services/aiDecision.ts`. Use `httpx` only for the later `call_ai_for_decision()` function.

- [ ] **Step 4: Run test to verify it passes**

Run: `cd backend_py && pytest tests/test_ai.py -v`
Expected: PASS.

### Task 7: Scheduler And WebSocket Snapshot Flow

**Files:**
- Create: `backend_py/app/services/scheduler.py`
- Create: `backend_py/app/ws.py`
- Modify: `backend_py/app/main.py`
- Create: `backend_py/tests/test_ws.py`

**Interfaces:**
- Consumes: repositories and services from earlier tasks.
- Produces: `start_scheduler() -> None`, `shutdown_scheduler() -> None`, `websocket_endpoint(websocket: WebSocket) -> None`.

- [ ] **Step 1: Write failing WebSocket connect test**

```python
from fastapi.testclient import TestClient

from backend_py.app.main import create_app


def test_websocket_accepts_ping():
    client = TestClient(create_app())
    with client.websocket_connect("/ws") as ws:
        ws.send_json({"type": "ping"})
        assert ws.receive_json()["type"] in {"pong", "snapshot"}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `cd backend_py && pytest tests/test_ws.py -v`
Expected: FAIL because `/ws` is missing.

- [ ] **Step 3: Implement minimal WebSocket and scheduler lifecycle**

Accept WebSocket connections, respond to ping, and provide snapshot messages using current DB reads. Start APScheduler on FastAPI startup and stop it on shutdown.

- [ ] **Step 4: Run test to verify it passes**

Run: `cd backend_py && pytest tests/test_ws.py -v`
Expected: PASS.

### Task 8: Runtime Switch

**Files:**
- Modify: `package.json`
- Create: `backend_py/requirements.txt`
- Modify: `Dockerfile`
- Modify: `docker-compose.yml` only if the runtime command or paths require it.

**Interfaces:**
- Consumes: `backend_py.app.main:app`.
- Produces: `pnpm run dev:backend` runs Python backend, Docker image builds frontend and runs Python backend.

- [ ] **Step 1: Write failing command check**

Run: `pnpm run dev:backend`
Expected: FAIL before script update because it starts the TypeScript backend.

- [ ] **Step 2: Update scripts and Docker**

Set `dev:backend` to `cd backend_py && uvicorn backend_py.app.main:app --reload --host 0.0.0.0 --port 5611`. Set backend build script to a Python syntax/import check. Change Docker runtime image to Python, install requirements, copy `backend_py`, and copy frontend build output into `backend_py/static`.

- [ ] **Step 3: Verify Python tests and frontend build**

Run: `cd backend_py && pytest -v`
Run: `pnpm --filter ./frontend build`
Expected: both PASS.

- [ ] **Step 4: Verify local server smoke**

Run: `cd backend_py && uvicorn backend_py.app.main:app --host 127.0.0.1 --port 5611`
In another shell: `curl http://127.0.0.1:5611/api/health`
Expected: JSON contains `"status":"healthy"`.

## Self-Review

- Spec coverage: schema, seed, routes, trading services, AI parsing, scheduler, WebSocket, scripts, and Docker each have a task.
- Placeholder scan: no implementation step depends on undefined placeholders.
- Type consistency: exported Python function names are consistent across task interfaces and tests.
