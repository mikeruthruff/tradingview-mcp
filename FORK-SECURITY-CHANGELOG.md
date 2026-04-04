# Fork Security Changelog

This document records all security-motivated modifications made to this fork
(`mikeruthruff/tradingview-mcp`) relative to upstream (`tradesdontlie/tradingview-mcp`).

These changes are permanent and must be preserved across upstream merges. When
cherry-picking from upstream, verify that none of these removals are reintroduced.

---

## 2026-04-03: Initial fork hardening

### 1. Removed `ui_evaluate` — arbitrary JS execution

**What:** Removed the `ui_evaluate` MCP tool, `tv ui eval` CLI command, and
`uiEvaluate()` core function.

**Why:** This tool accepted arbitrary JavaScript strings and executed them inside
TradingView's Electron renderer process via `Runtime.evaluate()`. While all other
tools construct their own JS from validated parameters, `ui_evaluate` passed
user/agent input directly — making it the widest attack surface. A prompt injection
or compromised agent could execute any code inside the TV session, including reading
session cookies, accessing broker connections, or manipulating the DOM.

**Files changed:**
- `src/core/ui.js` — removed `uiEvaluate()` function
- `src/tools/ui.js` — removed `ui_evaluate` MCP tool registration
- `src/cli/commands/ui.js` — removed `eval` subcommand

**Impact:** 77 tools remain. All other functionality is unaffected. The remaining
tools cover all documented use cases without requiring raw JS eval.

---

### 2. Removed `replay_trade` — simulated buy/sell/close in replay mode

**What:** Removed the `replay_trade` MCP tool and `tv replay trade` CLI command.
The core `trade()` function in `src/core/replay.js` is retained but unexposed.

**Why:** This tool calls `replayApi.buy()`, `replayApi.sell()`, and
`replayApi.closePosition()` inside TradingView's Electron process. While these are
replay-mode simulated trades (not live orders), they exercise the same internal
trading API surface that broker integrations use. TradingView allows users to connect
exchange accounts (Binance, Interactive Brokers, etc.) — making any code path that
touches trade execution APIs a high-value target for supply chain attacks. The
tradingview-mcp user base self-selects for traders who may have broker integrations
enabled, making this codebase an attractive target. Removing the trade execution
path eliminates this attack vector entirely. Replay observation tools (start, step,
autoplay, stop, status) are retained — we want to observe replays, not execute in them.

**Files changed:**
- `src/tools/replay.js` — removed `replay_trade` MCP tool registration
- `src/cli/commands/replay.js` — removed `trade` subcommand

---

### 3. Removed `depth_get` — order book / DOM data

**What:** Removed the `depth_get` MCP tool and `tv data depth` CLI command.

**Why:** Order book (Depth of Market) data is only relevant when a broker connection
is active. Exposing this tool normalizes interaction with broker-connected features
and widens the surface area toward execution infrastructure. It has no use case in
our analysis-only workflows. Removing it reinforces the principle that this fork is
strictly for chart analysis and indicator reading — not brokerage interaction.

**Files changed:**
- `src/tools/data.js` — removed `depth_get` MCP tool registration
- `src/cli/commands/data.js` — removed `depth` subcommand

---

### 4. Removed `trading` from `ui_open_panel` — broker panel access

**What:** Removed `'trading'` from the allowed panel enum in `ui_open_panel`.

**Why:** The Trading Panel is where broker connections, order entry, and position
management live in TradingView. Combined with the UI automation tools (`ui_click`,
`ui_type_text`, `ui_keyboard`, `ui_mouse_click`), an agent with access to the
Trading Panel could: open the panel, navigate to a broker connection, type order
quantities, and submit orders. Even without `ui_evaluate`, the remaining UI automation
tools provide sufficient building blocks for order execution if the Trading Panel is
accessible. Removing it from the allowed enum means `ui_open_panel` will reject
any attempt to open it with a Zod validation error — not a runtime check that could
be bypassed.

**Files changed:**
- `src/tools/ui.js` — removed `'trading'` from panel enum

---

### 5. Hardened npm supply chain

**What:** Added `.npmrc` with `ignore-scripts=true` and pinned exact dependency
versions (removed `^` caret ranges).

**Why:** The MCP SDK pulls in 91 transitive packages (Express, Hono, Zod, auth libs,
etc.). The `ignore-scripts` setting blocks `postinstall`/`preinstall`/`prepare`
lifecycle scripts — the #1 npm supply chain attack vector. Exact version pinning
prevents `npm install` from silently upgrading to a compromised release even if
`package-lock.json` is missing or regenerated.

**Files changed:**
- `.npmrc` — created with `ignore-scripts=true`
- `package.json` — pinned `@modelcontextprotocol/sdk` to `1.27.1`, `chrome-remote-interface` to `0.33.3`

---

## Tool inventory after hardening

**Removed (4):** `ui_evaluate`, `replay_trade`, `depth_get`, `trading` panel access

**Retained (74):** All chart reading, indicator reading, Pine Script development,
screenshot capture, alert management, watchlist management, replay observation,
batch operations, UI automation (excluding trading panel), and multi-pane layout tools.

---

## Upstream merge checklist

When cherry-picking commits from `tradesdontlie/tradingview-mcp`:

1. `git diff` the incoming changes against this changelog
2. Reject any commit that reintroduces a removed tool or capability
3. If `package.json` is modified: re-pin versions, diff the lock file, audit new transitive deps
4. If new tools are added: evaluate whether they touch trading/execution/broker APIs
5. Run `npm audit` after any dependency change
6. Never remove or modify `.npmrc`
