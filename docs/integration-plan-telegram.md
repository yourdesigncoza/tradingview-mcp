# TradingView MCP × claude-code-telegram Integration Plan

## Context

Trades Forex + XAUUSD on the London session for a prop firm. Goal: remote control of TradingView Desktop from phone via Telegram. Current state: TradingView MCP fully functional locally via Claude Code. Adding mobile capability via Telegram bot, plus scheduled morning-brief automation.

The chosen Telegram bot is `RichardAtCT/claude-code-telegram` (Python, MIT-licensed, actively maintained). It uses `claude-agent-sdk` to spawn Claude per request, loads MCP servers from a single JSON config, includes a FastAPI webhook receiver, and has APScheduler built in.

---

## Architecture

```
[Phone: Telegram app]
        │  HTTPS long-poll
        ▼
[Telegram cloud]
        │
        ▼
[Linux desktop @ home — same machine as TradingView]
 ├── systemd --user: tradingview-desktop.service
 │     └─ launch_tv_debug_linux.sh → TradingView Desktop → CDP :9222
 │
 ├── systemd --user: claude-code-telegram.service
 │     └─ python -m src.main (long-poll Telegram, FastAPI :8080)
 │           └─ per-request: node tradingview-mcp/src/server.js (stdio)
 │                                              └─ ws://localhost:9222
 │
 └── (optional M6) cloudflared tunnel exposing :8080 for TradingView alerts
```

**Co-location is mandatory** — TradingView MCP connects to CDP on `localhost:9222`, so the bot must run on the same machine where TradingView Desktop runs. No remote VPS deployment.

---

## Critical Files

**Existing (read-only references):**
- `src/server.js` — MCP entry (stdio)
- `src/core/index.js` — re-exports all 16 domain modules
- `scripts/launch_tv_debug_linux.sh` — already has Wayland XWayland fix
- `.mcp.json` — current Claude Code config (gitignored)

**Already created (ahead of execution):**
- `/home/laudes/.config/systemd/user/tradingview-desktop.service` — ✓ exists
- `/home/laudes/.config/systemd/user/claude-code-telegram.service` — ✓ exists
- `rules.json` — ✓ exists at project root

**To create:**
- `/home/laudes/zoot/projects/claude-code-telegram/` — clone target for bot repo
- `/home/laudes/zoot/projects/claude-code-telegram/.env` — bot configuration
- `/home/laudes/.config/claude-code-telegram/mcp.json` — MCP config (outside repo, survives reclone)
- `docs/integration-plan-telegram.md` — this file

---

## Configuration File Contents

### MCP config — `/home/laudes/.config/claude-code-telegram/mcp.json`

```json
{
  "mcpServers": {
    "tradingview": {
      "command": "node",
      "args": ["/home/laudes/zoot/projects/tradingview-mcp/src/server.js"],
      "env": {}
    }
  }
}
```

### Bot env — `/home/laudes/zoot/projects/claude-code-telegram/.env`

```bash
# Telegram
TELEGRAM_BOT_TOKEN=<from-@BotFather>
TELEGRAM_BOT_USERNAME=<bot-username>
ALLOWED_USERS=<user-numeric-id>

# Claude
ANTHROPIC_API_KEY=<sk-ant-...>
CLAUDE_MODEL=claude-sonnet-4-6
CLAUDE_MAX_TURNS=15

# MCP
ENABLE_MCP=true
MCP_CONFIG_PATH=/home/laudes/.config/claude-code-telegram/mcp.json

# Tool allowlist — READ-ONLY for v1 (Tier 0/1)
CLAUDE_ALLOWED_TOOLS=mcp__tradingview__chart_get_state,mcp__tradingview__chart_set_symbol,mcp__tradingview__chart_set_timeframe,mcp__tradingview__data_get_study_values,mcp__tradingview__data_get_ohlcv,mcp__tradingview__data_get_pine_lines,mcp__tradingview__data_get_pine_labels,mcp__tradingview__data_get_pine_tables,mcp__tradingview__data_get_pine_boxes,mcp__tradingview__quote_get,mcp__tradingview__symbol_info,mcp__tradingview__symbol_search,mcp__tradingview__capture_screenshot,mcp__tradingview__watchlist_get,mcp__tradingview__chart_manage_indicator,mcp__tradingview__indicator_set_inputs,mcp__tradingview__layout_list,mcp__tradingview__layout_switch,mcp__tradingview__tab_list,mcp__tradingview__tab_switch,mcp__tradingview__alert_list,mcp__tradingview__tv_health_check,mcp__tradingview__batch_run,mcp__tradingview__replay_start,mcp__tradingview__replay_step,mcp__tradingview__replay_status,mcp__tradingview__replay_stop

# Sandbox
APPROVED_DIRECTORY=/home/laudes/zoot/projects/tradingview-mcp

# Webhook receiver — DEFERRED to v2
ENABLE_API_SERVER=false

# Scheduler (M5)
ENABLE_SCHEDULER=true
TIMEZONE=Europe/London
NOTIFICATION_CHAT_IDS=<user-numeric-id>
```

### `rules.json` — project root

```json
{
  "watchlist": ["FOREXCOM:XAUUSD", "OANDA:EURUSD", "OANDA:GBPUSD", "OANDA:USDJPY", "OANDA:AUDUSD"],
  "timeframes": ["D", "240", "60"],
  "default_indicators": ["Relative Strength Index", "Moving Average Exponential", "Bollinger Bands"],
  "bias_thresholds": { "rsi_overbought": 70, "rsi_oversold": 30, "ema_fast": 21, "ema_slow": 50 },
  "session": { "name": "London", "open_utc": "07:00", "close_utc": "16:00" }
}
```

### systemd units

`/home/laudes/.config/systemd/user/tradingview-desktop.service`:
```ini
[Unit]
Description=TradingView Desktop with CDP
After=graphical-session.target
PartOf=graphical-session.target

[Service]
Type=simple
ExecStart=/home/laudes/zoot/projects/tradingview-mcp/scripts/launch_tv_debug_linux.sh 9222
Restart=on-failure
RestartSec=5

[Install]
WantedBy=graphical-session.target
```

`/home/laudes/.config/systemd/user/claude-code-telegram.service`:
```ini
[Unit]
Description=Claude Code Telegram bot (TradingView control)
After=network-online.target tradingview-desktop.service
Wants=network-online.target
BindsTo=tradingview-desktop.service

[Service]
Type=simple
WorkingDirectory=/home/laudes/zoot/projects/claude-code-telegram
EnvironmentFile=/home/laudes/zoot/projects/claude-code-telegram/.env
ExecStart=/home/laudes/zoot/projects/claude-code-telegram/.venv/bin/python -m src.main
Restart=on-failure
RestartSec=5

[Install]
WantedBy=default.target
```

**Note on `BindsTo=`:** if `tradingview-desktop.service` stops or crashes, the bot is also stopped — prevents the bot from silently answering against a dead TV instance.

---

## Workflow Scenarios

**1. Ad-hoc bias query** (`bias xauusd 4h`)
`chart_set_symbol → chart_set_timeframe → chart_get_state → data_get_study_values → data_get_pine_lines → quote_get → capture_screenshot → reply with text + PNG`

**2. Scheduled morning brief** (06:55 London weekdays)
APScheduler fires synthetic `/brief` → Claude reads `rules.json` → loops watchlist via `batch_run` → builds markdown table → sends with XAUUSD screenshot.

**3. TradingView alert webhook** (M6)
TV alert → `cloudflared` tunnel → `POST /webhooks/generic` (Bearer auth) → Claude pulls live confluence (study_values + pine_labels) → composes contextual message → Telegram.

**4. Replay data lab** (`replay xauusd 2024-03-15 step 50`)
`replay_start → loop replay_step + data_get_study_values → capture_screenshot → replay_stop (always) → CSV-style summary to Telegram`

---

## Security Model

**Tool allowlist tiers** (graduate over time):
- **Tier 0 (v1):** read-only data + screenshots
- **Tier 1 (v1):** chart navigation (set symbol/tf, switch layout, manage indicators) — included in v1 allowlist above
- **Tier 2 (later):** Pine writes (`pine_set_source`, `alert_create`) — gated behind `/confirm` flow
- **Tier 3 (future):** real-money execution — separate broker MCP with its own approval gate

**Excluded from v1:** `pine_set_source`, `pine_save`, `alert_create`, `alert_delete`, all `draw_*`, all `ui_*`, `tv_launch`, `tab_close/new`, `replay_trade`.

**Auth & exposure:**
- `ALLOWED_USERS` whitelist with single Telegram numeric ID
- Bot privacy mode ON in @BotFather
- Long-poll for Telegram (no inbound port for Telegram itself)
- `.env` never committed (gitignored)

**Defense-in-depth:**
- **Rate limiting:** tune token-bucket rate limiter to 6 req/min (default is 10)
- **Kill switch:** watch file at `/tmp/tv_bot_kill` + systemd path unit stops the bot immediately
- **Stolen-phone scenario:** Telegram 2FA on user account; no Tier 2+ tools without `/confirm <random-token>` flow

**Cost guards:** `CLAUDE_MAX_TURNS=15`, Sonnet for routine briefs. Estimated daily cost: ~$0.30 morning brief + ad-hoc.

---

## Build Sequence (Milestones)

### M0 — Pre-flight (15 min)
- Confirm screen lock disabled, screensaver off (TradingView Electron needs active rendering)
- Create Telegram bot via @BotFather, get token + numeric user ID
- Confirm Anthropic API key available
- **Note:** no linger — bot runs only while logged in. After reboot, log back in and systemd --user services start automatically.

### M1 — TradingView under systemd (30 min)
- Write `tradingview-desktop.service`, enable + start
- `systemctl --user status tradingview-desktop`
- Verify CDP via `curl http://localhost:9222/json/version`
- Reboot test: TV comes up after login

### M2 — Bot install + manual run (45 min)
- Clone `RichardAtCT/claude-code-telegram` to `/home/laudes/zoot/projects/`
- `python3.11 -m venv .venv && source .venv/bin/activate && pip install -e .`
- Write `.env` and `mcp.json` per above
- `python -m src.main` foreground
- Send `/start` from phone → expect welcome reply

### M3 — TV MCP wired through bot (30 min)
- From phone: "What's the price of EURUSD right now?"
- Expect Claude → `chart_set_symbol` + `quote_get` → reply with price
- Test `capture_screenshot` returns PNG to Telegram

### M4 — Bot under systemd + deep health check (45 min)
- Write `claude-code-telegram.service` with `BindsTo=tradingview-desktop.service`, enable + start
- `journalctl --user -u claude-code-telegram -f` to monitor
- Reboot test: bot responds within 30s of login
- **Add `/health` skill (deep check, mandatory):**
  1. Check `tradingview-desktop.service` is active
  2. Call `tv_health_check` MCP tool — verify CDP connection
  3. Call `quote_get` for EURUSD — verify sane price, not login-page error
  4. Reply with structured status; alert on any failure
- Add kill-switch path unit: stop bot when `/tmp/tv_bot_kill` exists
- Measure end-to-end latency for "price of X" — record baseline

### M5 — Morning brief on schedule (45 min)
- Add APScheduler job at 06:55 Europe/London weekdays
- Job posts synthesized brief prompt referencing `rules.json` watchlist
- Manual trigger first → verify output
- Wait for natural fire next weekday morning

### M6 — Hardening (ongoing)
- Add daily cost ceiling
- Tune rate limiter to 6 req/min
- Document recovery runbook (TV crash, login expiry, API key rotation)
- If M4 latency baseline is unacceptable (>5s for simple queries), evaluate HTTP transport for tradingview-mcp

---

## Verification

| Milestone | Command/Test | Expected | Failure Mode |
|---|---|---|---|
| M1 | `curl -s http://localhost:9222/json/version` | JSON with WebKit version | Path mismatch in launch script |
| M2 | `/start` from phone | Welcome reply | Token / ALLOWED_USERS wrong |
| M3 | "Price of EURUSD?" from phone | Price + symbol | MCP_CONFIG_PATH unresolved or `node src/server.js` exits non-zero |
| M4 | Reboot, wait 30s, send `/start` | Reply | Service not started or wrong WorkingDirectory |
| M4 (health) | `/health` from phone | Structured status with active service + valid quote | Login screen detected, dead service, or stale data |
| M4 (kill) | `touch /tmp/tv_bot_kill` then `/start` | No reply (bot stopped) | Bot still answers — path unit not loaded |
| M5 | Manual scheduler trigger | Markdown table + screenshot to TG | rules.json unread or batch_run perm missing |

---

## Open Risks

1. **Per-request MCP spawn latency** — claude-agent-sdk launches the MCP server fresh for each Telegram message. Expected overhead: 1-3s for Node startup + CDP connect. Acceptable for ad-hoc queries; measured at M4. v2 mitigation: HTTP transport for tradingview-mcp (persistent server).
2. **No linger** — bot dies at logout (user choice). Must log back in after reboot.
3. **Display lock / screensaver** — kills Electron rendering. GNOME/KDE settings must have auto-lock disabled.
4. **TradingView account session expiry** — TV Desktop occasionally logs out. Deep `/health` check detects this by validating actual quote data.
5. **Stolen phone / Telegram compromise** — read-only tier limits damage. Mitigations: Telegram 2FA, kill switch, no Tier 2+ tools without `/confirm`.
6. **Wayland flakiness** — handled via `--ozone-platform=x11` in launch script. Retest if DE changes.
7. **`MCP_CONFIG_PATH` env var name** — confirm exact spelling in `src/config/settings.py` at install time.
8. **Anthropic API cost drift** — set ceiling early, monitor first week.

---

## Out of Scope (v2+)

- **HTTP transport for tradingview-mcp** — eliminates per-request spawn latency. Trigger: M4 latency >5s.
- **Xvfb-isolated TradingView** — run TV Desktop in virtual framebuffer, survives screen lock/logout.
- **TradingView alert webhook receiver** — cloudflared tunnel + `/webhooks/generic` endpoint + Claude reasoning to Telegram.
- **Pre-trade risk MCP server** (prop-firm rule enforcement)
- **Broker MCP integration** (PickMyTrade or IB Gateway) with `/confirm` two-step
- **Tier 2 tools via Telegram** — `pine_set_source`, `alert_create`, etc.
- **Voice-message support** (Whisper transcription)
- **End-of-session report** (16:05 London cron with closed-trade analysis)
