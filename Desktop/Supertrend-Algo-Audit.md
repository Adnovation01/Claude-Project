# Supertrend Algo — Full Project Audit
**Location:** `C:\supertrend-algo`
**Audit date:** 2026-06-11
**Live VPS:** `138.252.201.204` (port 80)
**Version:** v2.0.0 (MT5 Edition)

---

## 1. What This Project Is

A multi-user Flask web application that bridges **TradingView** trade alerts to **MetaTrader 5 (MT5)** brokers (Exness, FundingPips, FundedNext, etc.). Each registered user connects their own MT5 account; the app receives webhook signals from TradingView and executes trades on that user's MT5 account in real time.

```
TradingView Pine Script Alert
        │  HTTP POST (JSON)
        ▼
Flask Web App (gevent WSGI, port 80)  ──  /tvwebhook
        │  Identifies user by "secret" key
        │  Parses signal (direction, lot size, magic number)
        ▼
Per-User MT5Worker Process (multiprocessing)
        │  MetaTrader5 Python API
        ▼
MT5 Terminal (on same VPS) → Broker Server → Trade Executed
```

---

## 2. Full File Structure

```
C:\supertrend-algo\
├── .env                        # Live secrets (REDACTED in this doc — see §8)
├── .env.template                # Template showing required env var names
├── .gitignore                   # Standard Python + Flask ignores; .env, instance/, logs excluded from old rules
├── execute_main.bat              # Windows: creates venv, installs requirements.txt, runs main.py
├── execute_main.sh               # Bash: creates venv_sh, installs requirements_sh.txt, runs main.py
├── main.py                       # Entry point — Flask + gevent server, MT5 worker bootstrap, background health loop
├── monitor.py                    # Standalone production health monitor (run by Task Scheduler every 1 min)
├── PINESCRIPT_ALERT_SETUP_PROMPT.txt   # Copy-paste prompt for AI tools to adapt Pine Script alerts to this bot's JSON format
├── README.md                     # User-facing setup/usage/troubleshooting documentation (352 lines)
├── requirements.txt              # Python deps for Windows (.bat) install — includes MetaTrader5 (Windows-only)
├── requirements_sh.txt           # Python deps for bash install — adds boto3, pandas, numpy (no MetaTrader5)
├── start_service.bat             # Minimal launcher used by the "SupertrendAlgo" scheduled task
├── instance/
│   └── database.db               # SQLite DB (24,576 bytes) — users, MT5 accounts, triggers, trades
├── logs/
│   ├── app.log                   # Flask application log (rotates at 10MB, 10 backups)
│   ├── monitor.log                # monitor.py log (rotates at 5MB, 1 backup as monitor_old.log)
│   ├── monitor_old.log            # Previous monitor log (5,242,966 bytes)
│   └── monitor_state.txt          # Single word: "up" or "down" — last known health state
├── venv_sh/                       # Bash-created virtual environment (present, populated)
├── utils/
│   ├── logger.py                  # Rotating file + console logger setup
│   ├── mt5_manager.py             # MT5Worker (per-user process) + MT5Manager (pool manager)
│   └── shared.py                  # Shared globals: colors, version, secret generator, SharedResourceManager
└── web/
    ├── __init__.py                # Flask app factory — extensions, blueprints, DB init, login manager
    ├── auth.py                    # Login, logout, register, create_super_user
    ├── models.py                  # SQLAlchemy models + WTForms (User, MT5Account, TradingviewTrigger, Trade)
    ├── tvviews.py                 # /health and /tvwebhook routes — the core trading logic
    ├── views.py                   # Dashboard, profile, MT5 connect/reconnect, trade log, /alert_webhook stub
    ├── static/
    │   ├── dashboard.css / dashboard.js
    │   ├── login.css
    │   ├── profile.css / profile.js
    │   └── register.css
    └── templates/
        ├── base.html              # Shared layout/nav
        ├── dashboard.html          # Webhook URL + alert template display
        ├── login.html
        ├── register.html
        ├── profile.html            # Profile + MT5 credential form
        └── trades.html             # Trade log with filters & pagination
```

---

## 3. How Each Component Works

### 3.1 `main.py` — Entry Point
- Creates a `gevent.pywsgi.WSGIServer` bound to `0.0.0.0:80` with `WebSocketHandler` (for Flask-SocketIO).
- `signal_handler()` — graceful shutdown on SIGINT (stops server, shuts down thread pool).
- `env_init()` — **first-run setup wizard**: if `.env` doesn't exist, interactively prompts for `REGISTER_SECRETKEY`, `SUPERUSER_USERNAME/NAME/PASSWORD`, `NGROK_ENABLED`, writes `.env`, then loads it.
- `init()` — runs console banner, env init, `create_app()`, `create_super_user()`.
- `bg_task_setup()` — spawns `foreverLoop` in a `ThreadPoolExecutor`.
- **`foreverLoop`** (background thread):
  - Sleeps 5s, then `load_all_users_mt5(reset=True)` — connects every user's saved MT5 account on startup.
  - Loops forever; every **60 seconds** calls `_health_check_all_workers()`.
- **`_health_check_all_workers()`**: for every `User` with an `mt5_account`, if `mt5_manager.is_connected(user.id)` is False, attempts `connect_user()` again and updates `MT5Account.status` to `connected`/`disconnected` in the DB.
- `open_browser()` — determines `public_url`:
  - If `NGROK_ENABLED=1`, opens an ngrok tunnel and uses that URL.
  - Otherwise uses `http://{PUBLIC_HOST}` where `PUBLIC_HOST` defaults to `138.252.201.204`.
- `main()` — `init()` → `bg_task_setup()` → `open_browser()` → `server.serve_forever()`.

### 3.2 `utils/mt5_manager.py` — MT5 Worker Pool

**`MT5Worker(multiprocessing.Process)`** — one dedicated OS process per user's MT5 connection:
- `__init__`: stores `user_id`, MT5 `login`/`password`/`server`, plus `cmd_queue`, `result_queue`, `connected` (Event), `connection_error` (shared Value).
- `run()`:
  1. Imports `MetaTrader5` *inside* the subprocess (required because the SDK is process-bound).
  2. `mt5.initialize()` then `mt5.login(...)`. On failure, sets `connection_error=1` and returns.
  3. Sets `connected` event.
  4. Main loop: blocks on `cmd_queue.get(timeout=60)`.
     - On timeout (60s idle) → **keepalive check**: if `mt5.terminal_info()` is `None` (connection lost), calls `mt5.shutdown()` then re-`initialize()` + re-`login()`.
     - `STOP` → breaks loop, `mt5.shutdown()` in `finally`.
     - `PLACE_ORDER` → `_place_order()`, result pushed to `result_queue`.
     - `CLOSE_POSITION` → `_close_position()`, result pushed to `result_queue`.

- **`_place_order(mt5, cmd)`**:
  - `mt5.symbol_select(symbol, True)` to ensure visibility in Market Watch.
  - Gets tick + symbol info; fails cleanly if either is `None`.
  - Picks `ask` price for longs, `bid` for shorts.
  - **Auto-detects filling mode**: prefers `ORDER_FILLING_IOC`, falls back to `FOK`, then `RETURN` — handles broker differences automatically.
  - Sends a `TRADE_ACTION_DEAL` market order with `deviation=20`, the alert's `magic` number, comment `"TV signal"`, `type_time=ORDER_TIME_GTC`.
  - Returns `success`, `ticket`, `retcode`, `comment`, `price`, `error`.

- **`_close_position(mt5, cmd)`**:
  - `mt5.positions_get(symbol=symbol)`, filters to positions matching the alert's `magic` number — **this is how exits target only the correct strategy's trade** even if multiple strategies trade the same symbol.
  - For each matching position, sends an opposite-direction `TRADE_ACTION_DEAL` referencing `position=pos.ticket`, comment `"TV exit"`.
  - Returns per-ticket close results plus an overall `success` flag.

**`MT5Manager`** — pool manager (module-level singleton `mt5_manager`):
- `connect_user()` — starts a new `MT5Worker` (or reuses a live one), waits up to **15s** for `connected` event, terminates the worker on timeout/login failure.
- `place_order()` / `close_position()` — put a command on `cmd_queue`, block on `result_queue.get(timeout=30)`; on timeout returns a `success: False` error result.
- `disconnect_user()` — sends `STOP`, joins with 5s timeout, force-`terminate()`s if still alive.
- `is_connected()` — `True` if the worker process object exists and `is_alive()`.
- `reconnect_user()` — disconnect then connect.

**Module-level functions:**
- `load_user_mt5(user_id)` — Flask-Login's `user_loader`; on every request, if the user has saved MT5 creds and isn't connected, connects them.
- `load_all_users_mt5(reset=False)` — connects/reconnects every user with saved MT5 creds (used at startup with `reset=True`).

### 3.3 `web/__init__.py` — Flask App Factory
- Globals: `app` (Flask), `db` (SQLAlchemy), `bcrypt`, `socketio` (SocketIO), `login_manager`.
- `APP_PORT = 80`.
- `create_app(load_user_mt5)`:
  - `SQLALCHEMY_DATABASE_URI = 'sqlite:///database.db'`
  - `SECRET_KEY = os.getenv('FLASK_SECRET_KEY', 'xK9#mP2$vL7@nQ4&')` — **hardcoded fallback secret** (see §10 risks).
  - `CORS(app, origins="*")` — wide open CORS.
  - Registers blueprints: `auth` (`/`), `views` (`/`), `tvviews` (`/`).
  - `socketio.init_app(..., async_mode='gevent', cors_allowed_origins="*", transports=['websocket'])`.
  - `db.create_all()` via `create_database()`.
  - `login_manager.login_view = 'auth.login'`; `user_loader` delegates to `load_user_mt5`.

### 3.4 `web/auth.py`
- `/login` (GET/POST) — `LoginForm`, bcrypt password check, `login_user()`, redirects to dashboard.
- `/logout` — `login_required`, `logout_user()`, redirect to login.
- `/register` (GET/POST) — `RegisterForm`; requires `secretKey` to match `REGISTER_SECRETKEY` env var; on success, hashes password with bcrypt, generates a unique 12-char alphanumeric `tv_secret`, creates `User(role='user')`.
- `create_super_user()` — called once at startup; if **no users exist at all**, creates a `super_user` from `SUPERUSER_USERNAME/NAME/PASSWORD` env vars with its own generated `tv_secret`.

### 3.5 `web/tvviews.py` — Core Trading Logic
- Module tracks `_start_time = datetime.now()` at import time for uptime reporting.

**`GET /health`**:
```json
{
  "status": "ok",
  "uptime": "Xh Ym Zs",
  "total_trades": <count>,
  "timestamp": "<iso datetime>"
}
```
Returns HTTP 200 always (does not check MT5 connectivity or DB health beyond a `Trade.query.count()`).

**`GET/POST /tvwebhook`** — the heart of the bot:
1. Logs raw request data.
2. Parses JSON body → 400 `{"status":"failed","error":"invalid json"}` on failure.
3. Extracts `secret`, `ticker`, `volume`, `magic`, `alert_message`.
4. Validates all 5 fields present → 400 if any missing.
5. Validates `volume` is a float ≥ 0.01 and `magic` is an int → 400 on failure.
6. **Always logs a `TradingviewTrigger` row first** (audit trail), regardless of what happens next.
7. Looks up `User` by `tv_secret == secret`. If not found → `status='failed'`, logs reason, still returns HTTP 200 `{"status":"success"}` (note: webhook always returns 200 to avoid TradingView retry storms, but internal `status` field / DB record reflects the real outcome).
8. Checks `mt5_manager.is_connected(user.id)` — if not connected, fails with a message telling the user to configure MT5 in Profile.
9. Parses `alert_message`:
   - `is_long = 'long' in alert_message`
   - `is_entry = 'entry' in alert_message`
   - `exit_reason` = 3rd word if present (e.g., `SL`, `TP`, or custom like `TRAIL`)
10. **Entry path**: `pool.spawn(mt5_manager.place_order, ...).get(timeout=35)` (gevent threadpool — non-blocking for other requests). On success, creates a `Trade` row with `entry_exit='entry'`, `entry_price`, `mt5_ticket`.
11. **Exit path**: `pool.spawn(mt5_manager.close_position, ...).get(timeout=35)`. On success, creates one `Trade` row per closed position with `entry_exit='exit'`, `exit_price`, `exit_reason`.
12. Updates the `TradingviewTrigger.status` to `'complete'` or `'failed'` and commits.
13. **Always returns HTTP 200** `{"status": "success"}` regardless of internal outcome.

### 3.6 `web/views.py`
- `MT5_ALERT_TEMPLATE` — JSON template string shown on the dashboard, pre-filled with the user's `tv_secret`.
- `/` (`dashboard`, login required) — renders webhook URL (`{public_url}/tvwebhook`) and the alert template.
- `/profile` (login required):
  - `ProfileForm` — update name/email.
  - `MT5ApiForm` (prefix `mt5`) — create or update `MT5Account` (login, password, server, broker), then immediately calls `mt5_manager.reconnect_user()` and updates `status` to `connected`/`disconnected` based on result, with flash messages.
- `/mt5_reconnect` (login required) — re-runs `reconnect_user()` using saved credentials; flashes result.
- `/trades` (login required) — paginated (25/page) trade log filterable by `symbol` (ILIKE), `direction`, `entry_exit`, ordered newest-first.
- `/alert_webhook` (POST) — stub endpoint, returns plain text `"test"` (unused/legacy).

### 3.7 `utils/shared.py`
- ANSI color shortcuts via `colorama` (`lg`, `w`, `cy`, `ye`, `r`, `n`, `mg`).
- `VERSION = "v2.0.0"`.
- `handleYorN()` — converts y/n/true/1/yes style input to 1/0.
- `generate_alphanumeric_secret(length=12)` — random alphanumeric string for `tv_secret`.
- `custom_round(value, base=.05, prec=2)` — rounding helper (currently unused in the visible flow).
- `SharedResourceManager` — holds `logger_global` (from `logger_setup()`) and `public_url`. Instantiated once as `shared_obj`.

### 3.8 `utils/logger.py`
- Logger name: `supertrend-algo-bot`.
- Writes to `./logs/app.log`.
- `RotatingFileHandler(maxBytes=10*1024*1024, backupCount=10)` — rotates at 10MB, keeps 10 backups (up to ~110MB of history).
- Also logs to console (`StreamHandler`).
- Format: `%(asctime)s - %(levelname)s - %(filename)s:%(lineno)d - %(message)s`.

---

## 4. Database Schema (`instance/database.db`, SQLite)

### `users`
| Column | Type | Notes |
|---|---|---|
| id | Integer PK | |
| username | String(20) | unique, not null |
| name | String(80) | not null |
| email | String(80) | nullable |
| password | String(80) | bcrypt hash, not null |
| role | String(20) | `super_user` or `user` |
| tv_secret | String(12) | unique-ish 12-char secret used to identify webhook source |

Relationships: `mt5_account` (1:1), `tv_triggers` (1:many), `trades` (1:many)

### `mt5_accounts`
| Column | Type | Notes |
|---|---|---|
| id | Integer PK | |
| user_id | FK → users.id | |
| login | Integer | MT5 account number |
| password | String(100) | **plaintext MT5 password** |
| server | String(100) | e.g. `Exness-MT5Real8` |
| broker | String(50) | display name |
| status | String(20) | `connected` / `disconnected`, default `disconnected` |

### `tradingview_triggers`
| Column | Type | Notes |
|---|---|---|
| id | Integer PK, autoincrement | |
| user_id | FK → users.id | nullable (set after lookup) |
| secret | String(12) | not null — raw secret as received |
| ticker | String(30) | not null |
| trigger_price | Float | nullable, currently unused |
| volume | Float | not null |
| magic | Integer | nullable |
| alert_message | String(80) | not null |
| status | String(20) | `received` → `complete`/`failed` |
| datetime | DateTime | default `datetime.now` |

Relationships: `user` (many:1), `trade` (1:1, `uselist=False`)

### `trades`
| Column | Type | Notes |
|---|---|---|
| id | Integer PK, autoincrement | |
| user_id | FK → users.id | |
| tv_trigger_id | FK → tradingview_triggers.id | |
| symbol | String(30) | |
| direction | String(10) | `long` / `short` |
| entry_exit | String(10) | `entry` / `exit` |
| volume | Float | |
| magic | Integer | |
| mt5_ticket | Integer | MT5 order ticket number |
| entry_price | Float | 0.0 for exit rows |
| exit_price | Float | 0.0 for entry rows |
| exit_reason | String(50) | e.g. `SL`, `TP`, custom, empty for entries |
| datetime | DateTime | default `datetime.now` |

### Forms (WTForms, in `models.py`)
- `LoginForm` — username (4-20 chars), password (8-20 chars)
- `RegisterForm` — username, name, password, secretKey; validates username uniqueness
- `ProfileForm` — name, email
- `MT5ApiForm` — login (int ≥1), password, server, broker

### Current Live Data (as of audit)
| id | username | role | MT5 broker | MT5 server | MT5 status |
|---|---|---|---|---|---|
| 1 | admin | super_user | — | — | not configured |
| 2 | nalinpatel | user | — | — | not configured |

- **Total trades:** 0
- **Total triggers:** 0

→ The system is deployed and running but **no MT5 account has been connected yet and no trades have executed**. This is a fresh/test deployment.

---

## 5. TradingView Integration Contract

### Alert JSON format
```json
{
  "secret": "YOUR_TV_SECRET",
  "ticker": "XAUUSD",
  "volume": 0.01,
  "magic": 1001,
  "alert_message": "long entry"
}
```

### `alert_message` values
| Value | Action |
|---|---|
| `long entry` | Buy / open long |
| `long exit` | Close long |
| `long exit SL` | Close long, logged with reason `SL` |
| `long exit TP` | Close long, logged with reason `TP` |
| `short entry` | Sell / open short |
| `short exit` | Close short |
| `short exit SL` / `short exit TP` | Close short with reason |

Any third word in `alert_message` becomes the `exit_reason` (supports custom reasons like `TRAIL`).

### Magic number system
- The `magic` number tags MT5 positions on entry.
- Exits search only for positions with the matching `magic`, so **multiple independent strategies can run on the same symbol simultaneously** without interfering with each other.
- **Rule enforced by convention (not code):** each strategy must use a unique magic number.

### `PINESCRIPT_ALERT_SETUP_PROMPT.txt`
A ready-made prompt block (12,794 bytes) meant to be pasted into an AI assistant along with a user's Pine Script, instructing the AI to modify the script's alerts to emit the exact JSON contract above for all 8 entry/exit/SL/TP combinations.

---

## 6. Deployment & Process Management

### `execute_main.bat` (Windows)
1. `python -m venv venv`
2. Activate venv
3. `pip install -r requirements.txt`
4. `start cmd /k ... python main.py` (opens in new console window)

### `execute_main.sh` (bash)
1. `python3 -m venv venv_sh`
2. Activate venv_sh
3. `pip install -r requirements_sh.txt`
4. `python3 main.py` (foreground)

### `start_service.bat` (used by Task Scheduler)
```bat
@echo off
cd /d C:\supertrend-algo
"C:\Program Files\Python311\python.exe" main.py
```
Minimal — no venv activation, runs directly with the system Python 3.11 install. Implies dependencies are installed globally for `C:\Program Files\Python311\python.exe`, not in a venv, on the production VPS.

### Dependency files differ:
- `requirements.txt` (Windows/production): includes `MetaTrader5` (Windows-only package), no `boto3`/`pandas`/`numpy`.
- `requirements_sh.txt` (bash/dev): includes `boto3`, `pandas`, `numpy`, `jmespath`, `s3transfer` but **no `MetaTrader5`** — meaning the bash setup cannot actually place trades (consistent with MT5 being Windows-only).

---

## 7. Production Reliability / Uptime Architecture (3-Tier Watchdog)

This is the most significant operational component of the project — designed for ~99% uptime with full self-healing.

### Tier 1 — `SupertrendAlgo` (Scheduled Task)
- **Trigger:** On boot
- **Action:** runs `start_service.bat` → `main.py` → Flask app on port 80
- **Internal resilience:** `foreverLoop` thread (60s interval) reconnects dead per-user MT5 worker processes; each `MT5Worker` self-heals its MT5 connection on 60s idle if `terminal_info()` is lost.

### Tier 2 — `SupertrendAlgoMonitor` (Scheduled Task)
- **Trigger:** Every 1 minute (`PT1M`) + on boot
- **Action:** `python.exe C:\supertrend-algo\monitor.py`
- **Behavior** (`monitor.py`, 442 lines):
  1. **Log rotation**: rotates `monitor.log` → `monitor_old.log` at 5MB (deletes any existing old file first).
  2. **Health checks**:
     - `is_port_listening(80)` — raw TCP connect to `127.0.0.1:80`, 3s timeout.
     - `get_task_state()` — PowerShell `Get-ScheduledTask -TaskName 'SupertrendAlgo'` → `.State` (expects `Running` or `Ready`).
     - `get_recent_app_errors(lines=100)` — tails last 100 lines of `app.log`, filters for `ERROR`, `CRITICAL`, `Traceback`, `Exception`.
  3. **Known-error auto-fixes** (`apply_known_fix`), matched against `KNOWN_ERRORS` list:
     - `"database is locked"` → deletes `instance/database.db-wal` and `-shm` files.
     - `"address already in use"` / `"10048"` → PowerShell `Get-NetTCPConnection -LocalPort 80 | Stop-Process` to kill whatever is squatting on port 80.
     - `"ModuleNotFoundError"` / `"No module named"` → `pip install -r requirements.txt -q`.
     - `"REGISTER_SECRETKEY"` (missing `.env`) → regenerates a default `.env` with `REGISTER_SECRETKEY=secret@2026`, `SUPERUSER_USERNAME=admin`, `SUPERUSER_PASSWORD=admin`, `NGROK_ENABLED=n`, preserving `GEMINI_API_KEY` and `ALERT_EMAIL` from current env if set.
  4. **Unknown-error AI diagnosis** (`ask_gemini` + `apply_gemini_fix`):
     - If errors don't match any known signature, sends the last 30 error lines plus the **full source** of `main.py`, `web/__init__.py`, `web/models.py`, `web/tvviews.py`, `utils/mt5_manager.py` (truncated to 3000 chars each) to **Gemini 2.0 Flash** (`google.generativeai`).
     - Prompt asks Gemini to respond *only* in `FILE/FIND/REPLACE/---` or `COMMAND/---` blocks, or `NO_FIX`.
     - `apply_gemini_fix()` parses these blocks and:
       - For `COMMAND` blocks: runs the PowerShell command directly (60s timeout).
       - For `FILE/FIND/REPLACE` blocks: opens the file, does a single `str.replace(find, replace, 1)` if `find` is present, writes it back — **directly modifying production source code with no backup, diff review, or git commit**.
  5. **Restart logic** (`restart_app`): up to 3 attempts —
     - `Stop-ScheduledTask SupertrendAlgo` → wait 4s → `Start-ScheduledTask SupertrendAlgo`
     - Wait `10s, 15s, 20s` (attempt 1/2/3) then re-check `is_port_listening(80)`.
     - Returns `True` on first successful recovery, `False` if all 3 fail.
  6. **Email alerting** via Gmail SMTP (`smtp.gmail.com:587`, STARTTLS):
     - `alert_down(reason)` — sent only on state transition `up → down` (deduplicated via `monitor_state.txt`).
     - `alert_recovered()` — sent only on transition `down → up`.
     - `[CRITICAL] Supertrend Algo recovery FAILED` — sent if all 3 restart attempts fail; includes RDP IP and log paths for manual intervention.
  7. **Main flow** (`run()`): rotate log → check port/task/errors → apply known/AI fixes if needed → restart if `needs_restart` → alert accordingly → log "Health check end".

### Tier 3 — `SupertrendAlgoWatchdog` (Scheduled Task)
- **Trigger:** Every 3 minutes (`PT3M`) + on boot
- **Action:** PowerShell one-liner —
  ```powershell
  if ((Get-ScheduledTask -TaskName 'SupertrendAlgoMonitor').State -eq 'Ready') {
      Start-ScheduledTask -TaskName 'SupertrendAlgoMonitor'
  }
  ```
- Purpose: if the Monitor task itself stalls/crashes and sits idle ("Ready" without re-triggering), the Watchdog kicks it back into action. This closes the loop so even the monitoring layer is self-healing.

### Live Status at Audit Time
- `monitor_state.txt` = `up`
- `SupertrendAlgo` task state = `Running`
- Last 6+ consecutive 1-minute health checks: `Port 80: UP | Task: Running | Errors: 0` → `OK — app is healthy`

### Recent Incident History (from `monitor.log`, 2026-06-11)
| Time | Event | Outcome |
|---|---|---|
| 19:53:37 | ALERT: Port 80 not responding | Restart attempt 1 succeeded; recovered by 19:54:01 |
| 20:34:40 | ALERT: Port 80 not responding + Task state "Unknown" | Restart attempt 1 succeeded; recovered by 20:35:10 |
| 20:36:39 | ALERT: Port 80 not responding (again, ~2 min after prior recovery) | Restart attempt 1 succeeded; recovered by 20:37:06 |

`app.log` shows the app's startup banner printing 3 times between 20:34:35 and 20:36:54 — three full process restarts within ~2.5 minutes. All were auto-recovered on the **first** restart attempt (no Gemini AI escalation triggered — `app.log` had 0 ERROR/CRITICAL/Traceback lines at the time of the audit). The root cause of the brief flapping window was not captured in the logs (no error signatures present), so it cannot be conclusively diagnosed from current data — worth watching for recurrence.

---

## 8. Configuration (`.env`) — Keys Present (values redacted)

| Key | Purpose |
|---|---|
| `REGISTER_SECRETKEY` | Secret required to self-register a new user account |
| `SUPERUSER_USERNAME` | Initial admin username (used only if DB has zero users at startup) |
| `SUPERUSER_NAME` | Initial admin display name |
| `SUPERUSER_PASSWORD` | Initial admin password |
| `NGROK_ENABLED` | `y`/`n` — whether to open an ngrok tunnel for the webhook URL |
| `GEMINI_API_KEY` | Used by `monitor.py` for AI-based error diagnosis/auto-patching |
| `ALERT_EMAIL` | Destination address for monitor email alerts |
| `GMAIL_USER` | Gmail account used to *send* alert emails |
| `GMAIL_APP_PASS` | Gmail app password for SMTP auth |
| `FLASK_SECRET_KEY` | Flask session signing key (overrides the hardcoded fallback) |

`.env.template` only documents 5 of these 10 keys (`REGISTER_SECRETKEY`, `SUPERUSER_USERNAME`, `SUPERUSER_NAME`, `SUPERUSER_PASSWORD`, `NGROK_ENABLED`) — the monitoring/email/AI/Flask-secret keys were added later and aren't reflected in the template, so a fresh first-run setup wizard (`env_init()`) won't prompt for them.

---

## 9. Dependencies Summary

**Core web stack:** Flask 3.0.3, Flask-SQLAlchemy 2.0.31/3.1.1, Flask-Login, Flask-Bcrypt, Flask-WTF, Flask-CORS, Flask-SocketIO 5.3.6 + python-socketio/engineio.

**Async/realtime:** gevent 24.2.1, gevent-websocket, greenlet, simple-websocket, wsproto, websockets.

**Trading:** `MetaTrader5` (Windows-only, production requirements.txt only).

**Tunneling/UX:** pyngrok 7.1.6, pyfiglet (banner art), colorama (terminal colors), maskpass/pynput/getpass (credential prompts).

**Monitor-specific** (used directly via stdlib + installed packages): `subprocess`, `smtplib`, `socket`, `google.generativeai` (Gemini SDK — must be present for AI diagnosis to work; not listed in `requirements.txt`, so it likely needs separate installation on the VPS).

---

## 10. Security & Reliability Findings

| # | Finding | Severity | Detail |
|---|---|---|---|
| 1 | Hardcoded fallback `SECRET_KEY` (`'xK9#mP2$vL7@nQ4&'`) in `web/__init__.py` | High | Used if `FLASK_SECRET_KEY` env var unset. Predictable secret = forgeable session cookies. |
| 2 | `CORS(app, origins="*")` and SocketIO `cors_allowed_origins="*"` | Medium | Any website can make cross-origin requests to the API. |
| 3 | App runs on plain HTTP (port 80), not HTTPS | High | TradingView posts the user's `tv_secret` (and trade details) in cleartext over the public internet — interceptable. |
| 4 | MT5 account passwords stored in plaintext in SQLite (`mt5_accounts.password`) | High | Acknowledged in README's own Security Notes. DB file (`instance/database.db`) is a single point of credential compromise for every connected broker account. |
| 5 | Gemini AI can directly rewrite production source files (`FILE/FIND/REPLACE`) and run arbitrary PowerShell commands (`COMMAND`) with no human review, diff, or git commit before/after | High | A bad/hallucinated patch could be written straight to disk and then the app restarted on top of it — potentially compounding failures. Also bypasses the CLAUDE.md mandatory commit/push-after-every-change policy entirely. |
| 6 | `monitor.py`'s "database is locked" fix deletes WAL/SHM files rather than addressing root cause | Medium | SQLite isn't configured with WAL mode + `busy_timeout` at the app level (`web/__init__.py`), so multi-process writes (main app + worker callbacks) can still lock. The monitor treats the symptom repeatedly rather than the cause. |
| 7 | `is_port_listening` health check is a bare TCP connect | Medium | A hung/deadlocked gevent event loop that still accepts TCP connections (but never responds) would not be detected as "down" — `/health` endpoint would be a stronger check. |
| 8 | No database backup strategy observed | Medium | `instance/database.db` (users, MT5 creds, trade history) has no scheduled export/backup. |
| 9 | Single VPS, no failover | Medium | All 3 watchdog tiers run on the same machine (`138.252.201.204`); a VPS-level outage (host crash, network loss) defeats the entire self-healing stack. |
| 10 | `requirements.txt` does not list `google-generativeai` | Low | Required by `monitor.py`'s AI-diagnosis path but not pinned/installed via the documented requirements file — could silently no-op ("GEMINI_API_KEY init failed") if missing on a fresh VPS rebuild. |
| 11 | `.env.template` is out of date (5 of 10 required keys missing) | Low | First-run setup wizard won't collect monitoring/AI/email config, requiring manual `.env` editing post-setup. |
| 12 | `/alert_webhook` route returns plain `"test"` | Low | Looks like a leftover/legacy debug endpoint, unauthenticated, publicly reachable. |
| 13 | Brief 3x restart flap on 2026-06-11 (20:34–20:37) with no captured root-cause error | Low/Watch | Self-recovered each time on attempt 1; no error signature logged. Recommend deeper logging if it recurs. |

---

## 11. Git History (last 8 commits)
```
933e38a Fix stale dhan_api navbar reference, hardcoded secret key, missing Trade Log nav link
8c9934a Fix webhook URL to show public IP instead of localhost
e9ddd8a Upgrade to production monitoring: 1-min checks, email alerts, /health endpoint, watchdog
22a564f Switch app from port 8501 to port 80
0f95d87 Add trade log page with filters and pagination
a0ddfe0 Upgrade monitor with Gemini AI diagnosis and auto-fix
98c7953 Add local health monitor with auto-restart and self-healing
d3498bb Deploy supertrend-algo to VPS 138.252.201.204
```
Repo is on `main`, up to date with `origin/main`. Only untracked item: `logs/` directory (correctly excluded from version control — log files shouldn't be committed).

---

## 12. Summary — How 99% Uptime Is Actually Achieved

1. **Process-level restart on boot** (Tier 1 scheduled task) — survives VPS reboots automatically.
2. **1-minute active health polling** (Tier 2) checking port + task state + log errors.
3. **Automatic remediation** for the 4 most common real-world failure modes (DB lock, port conflict, missing dependency, missing config) without human intervention.
4. **AI-assisted remediation** for *novel* errors via Gemini, which can patch code and re-run commands.
5. **3-attempt restart with increasing backoff** (10s/15s/20s) before declaring a critical failure.
6. **Email alerting** on every state transition (down/recovered) and on total recovery failure, so a human is looped in only when automation can't fix it.
7. **A watchdog for the watchdog** (Tier 3, every 3 min) ensures the monitoring process itself can't silently die.
8. **Per-user process isolation** (MT5Worker) means one trader's MT5/broker issue can't crash the shared app or other traders' connections.
9. **In-process self-healing** (60s MT5 worker health check, 60s MT5 keepalive/reconnect inside each worker) handles transient MT5 terminal disconnects without needing a full app restart at all.

**Net effect:** most failures are caught and fixed within 1–4 minutes without any human action, and the system has multiple independent layers so that for any single layer to leave the app down, two or more layers must fail simultaneously.
