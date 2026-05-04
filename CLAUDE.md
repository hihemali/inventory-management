# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

Factory Inventory Management System Demo - Full-stack application with Vue 3 frontend, Python FastAPI backend, and in-memory mock data (no database).

## Critical Tool Usage Rules

### Subagents
- **vue-expert**: **MANDATORY** for creating or significantly modifying any `.vue` file
- **code-reviewer**: Use after writing significant code
- **Explore**: Use for codebase exploration and pattern searches
- **general-purpose**: Use for complex multi-step tasks

### Skills
- **backend-api-test** skill: Use when writing or modifying tests in `tests/backend/`

### MCP Tools
- **ALWAYS use GitHub MCP tools** (`mcp__github__*`) for ALL GitHub operations
  - Exception: Local branches only — use `git checkout -b`
- **ALWAYS use Playwright MCP tools** (`mcp__playwright__*`) for browser testing
  - Frontend: `http://localhost:3000` | API: `http://localhost:8001`

## Stack
- **Frontend**: Vue 3 + Composition API + Vite (port 3000)
- **Backend**: Python FastAPI (port 8001)
- **Data**: JSON files in `server/data/` loaded at startup via `server/mock_data.py` (singleton)

## Commands

```bash
# Backend — first time setup
cd server && uv venv && uv sync

# Backend — run
cd server && uv run python main.py

# Frontend
cd client && npm install && npm run dev

# Frontend production build (output: client/dist/)
cd client && npm run build

# All backend tests
cd tests && uv run pytest -v

# Single test file
cd tests && uv run pytest backend/test_inventory.py -v

# Single test
cd tests && uv run pytest backend/test_dashboard.py::test_summary_with_warehouse_filter -v

# With coverage
cd tests && uv run pytest --cov=../server --cov-report=html
```

## Architecture

### Data Flow
Vue filter state (`useFilters.js`) → `api.js` Axios calls with query params → FastAPI endpoint → `apply_filters()` / `filter_by_month()` in `mock_data.py` → Pydantic-validated response → Vue computed properties render derived data.

### Filter System
Four global filters managed by the `useFilters` composable (singleton refs shared across all views):
- **Time Period** (`selectedPeriod`): supports `YYYY-MM` and `Q1-2025` quarter format
- **Warehouse** (`selectedLocation`): multi-warehouse
- **Category** (`selectedCategory`): product category
- **Order Status** (`selectedStatus`): order state

Inventory endpoints do **not** support `month` filtering (inventory has no time dimension).

### Frontend Patterns
- Raw data in `ref()` (`allOrders`, `inventoryItems`), derived data in `computed()`
- Data loading pattern: `onMounted` → async API call → assign to ref → computed properties react
- Component communication: props down, `$emit` events up — never mutate props directly
- Charts are custom SVG (no chart library)
- Global styles in `App.vue`; component styles are scoped

### Backend Patterns
- All data loaded once at startup from `server/data/*.json` into memory
- `apply_filters(items, warehouse, category, status)` and `filter_by_month(items, month)` are the two shared filter helpers in `mock_data.py`
- When adding a new endpoint: define Pydantic model → add endpoint in `main.py` → add tests in `tests/backend/`
- CORS is open for all origins (demo only)

### Key API Endpoints
| Endpoint | Filters |
|---|---|
| `GET /api/inventory` | warehouse, category |
| `GET /api/orders` | warehouse, category, status, month |
| `GET /api/dashboard/summary` | warehouse, category, status, month |
| `POST /api/orders/{order_id}/purchase-order` | — |
| `GET /api/demand`, `/api/backlog` | none |
| `GET /api/spending/summary|monthly|categories|transactions` | none |
| `GET /api/reports/quarterly`, `/api/reports/monthly-trends` | none |

### Internationalization
`useI18n` composable wraps `locales/en.js` and `locales/ja.js`. The `LanguageSwitcher` component toggles locale globally.

### Testing
Tests use FastAPI `TestClient` (no running server needed). Fixtures in `conftest.py` provide `client`, `sample_inventory_item`, and `sample_order`.

## Code Style
- Document non-obvious logic changes with inline comments explaining the **why**

## Common Pitfalls
1. Use unique keys in `v-for` (`sku`, `order_id`, `month`) — never `index`
2. Validate dates before calling `.getMonth()` — data may have nulls
3. Update Pydantic models in `main.py` when changing JSON data structure
4. Revenue goal constants: $800K/month (single month), $9.6M YTD (all months)

## Design System
- Colors: Slate/gray (`#0f172a`, `#64748b`, `#e2e8f0`)
- Status colors: green / blue / yellow / red
- No emojis in UI
