# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

Factory Inventory Management System — full-stack demo with Vue 3 frontend, Python FastAPI backend, and in-memory mock data (no database).

## Commands

### Backend
```bash
cd server
uv run python main.py          # Start server on http://localhost:8001
                               # API docs at http://localhost:8001/docs
```

### Frontend
```bash
cd client
npm install && npm run dev     # Start dev server on http://localhost:3000
npm run build                  # Production build
```

### Tests
```bash
cd tests
uv run pytest backend/ -v                        # Run all backend tests
uv run pytest backend/test_inventory.py -v       # Run a single test file
uv run pytest backend/test_inventory.py::test_name -v  # Run a single test
```

## Architecture

**Stack**: Vue 3 + Composition API + Vite (port 3000) → FastAPI (port 8001) → JSON files in `server/data/`

**Data flow**: Global filters in `client/src/composables/useFilters.js` → `client/src/api.js` (Axios) → FastAPI query params → in-memory filtering in `server/main.py` → Pydantic validation → Vue computed properties

**Filter system**: 4 shared filters (Time Period, Warehouse, Category, Order Status) stored as module-level refs in `useFilters.js`, shared across all views without a Vuex/Pinia store. Filter changes trigger `watch` in each view to reload data.

**Mock data**: Loaded once at server startup from `server/data/*.json` via `server/mock_data.py`. Changes don't persist — restart server to reload. Pydantic models in `server/main.py` must match JSON structure.

**Tests**: pytest + FastAPI TestClient. `tests/backend/conftest.py` adds `server/` to sys.path and provides `client` fixture. Run from `tests/` directory (pytest.ini sets `testpaths = backend`).

## Code Style

- Always document non-obvious logic changes with comments

## Key Patterns

**Reactivity**: Raw data in `ref()` (`allOrders`, `inventoryItems`), derived/filtered data in `computed()`. Never mutate computed properties or props directly.

**API filtering**: Pass `'all'` to skip a filter; endpoints check `if param and param != 'all'` before filtering. Inventory has no month filter (no time dimension).

**v-for keys**: Always use unique IDs (`sku`, `order_number`), never array index.

**Date handling**: Always validate before calling `.getMonth()` — check `!isNaN(date.getTime())`.

## Tool Usage Rules

### Subagents
- **vue-expert**: MANDATORY for creating or significantly modifying any `.vue` file
- **code-reviewer**: Use after writing significant code
- **Explore**: Use for codebase exploration and pattern searches
- **backend-api-test** skill: Use when writing/modifying tests in `tests/backend/`

### MCP Tools
- **GitHub MCP** (`mcp__github__*`): Use for ALL GitHub operations (exception: local branches — use `git checkout -b`)
- **Playwright MCP** (`mcp__playwright__*`): Use for all browser testing against `http://localhost:3000` / `http://localhost:8001`

## Common Issues

1. Update Pydantic models in `server/main.py` whenever `server/data/*.json` structure changes
2. Revenue goals: $800K/month single warehouse, $9.6M YTD all warehouses
3. Inventory endpoint does not support `month` filter
4. Consistent category names required across all data files — `inventory.json` is the source of truth

## Design System

- Colors: Slate/gray (`#0f172a`, `#64748b`, `#e2e8f0`); status: green/blue/yellow/red
- Charts: Custom SVG with computed properties; CSS Grid for layouts
- No emojis in UI
- Currency formatting: `toLocaleString('en-US', { style: 'currency', currency: 'USD' })`
