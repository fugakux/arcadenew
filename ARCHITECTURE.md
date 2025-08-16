# Arcade — Architecture & Developer Handoff

_Last reviewed: Aug 16, 2025_

## 0) What this project is

A Discord-login arcade with:

- **Flask + Flask-SocketIO** backend (Eventlet worker) for REST endpoints and real-time games.
- **MongoDB** data store accessed via `pymongo`.
- **React (CRA)** frontend using `socket.io-client`, MUI, and Chart.js.
- Games implemented over **Socket.IO namespaces**:
  - Crash (`/crash`)
  - Coin Flip (`/coin`)
  - Mines (`/mines`)

All user balances are in **points** (virtual credits). There’s a basic **shop** (primary & secondary/marketplace), **vouchers**, **deposit** flow (with Solana-style signature), **activity log**, simple **admin** endpoints, and **“arcade wallet”** management.

---

## 1) Repo layout

```
/backend
  auth.py
  block_processor.py
  config.py
  data.py
  points.py
  socketio_events.py
  utils.py
  /endpoints
    activity.py
    admin.py
    arcade_wallet.py
    deposit.py
    request_spot.py
    secondary_shop.py
    shop.py
    user.py
    voucher.py
    wallet.py
  /games
    crash_game.py
    coin_game.py
    mines_game.py

/interface
  package.json
  public/...
  src/
    App.js, App.css
    components/
      CrashGame.js/.css
      CoinFlip.js/.css
      Minesweeper.js/.css
      BlockExplorer.js/.css
      Deposit.js/.css
      DiscordCallback.js
      Header.jsx/.css
      Login.js/.css
      Profile.js/.css
      RPSGame.js/.css (stubbed)
      Shop.js/.css
      casino.jsx (arena/home)
      Status.jsx (basic)
```

_No tests yet._

---

## 2) Backend

### 2.1 Stack & entrypoint

- **Framework:** Flask + Flask-CORS + Flask-SocketIO  
- **WS transport:** Eventlet (`eventlet.monkey_patch()`), SocketIO server  
- **Run file:** `backend/main.py` — creates the Flask app, applies CORS, mounts REST endpoints, registers Socket.IO namespaces, then kicks off a background **block processor** thread.  
- **Default run:** `socketio.run(app, host='0.0.0.0', port=8000)`

### 2.2 Config (`backend/config.py`)

- `SECRET_KEY`: Flask secret (replace in prod).
- `ALLOWED_ORIGINS`: permitted frontend origins for CORS.
- **Block time:** `BLOCK_TIME = 1` (used by block processor).
- Paths reference **Mongo**, and legacy file-paths are present but live data is via MongoDB in `data.py`.
- **Global `lock`** (threading.Lock) guards writes.

### 2.3 Data layer (`backend/data.py`)

- **Env:** loads `auth.env`
  - `MONGO_URI`
  - `MONGO_DB` (default `arcadeAppDB`)
- **Collections (by usage):**
  - `users`, `shop_items`, `secondary_shop`, `requests`, `vouchers`,
    `casino_stats`, `deposits`, `wallets`, `arcade_wallets`, `blocks`.
- **Helpers (selected):**
  - Load/save primitives: users, shop items, requests, vouchers, secondary shop, casino stats, deposits, wallets, arcade wallets, current block.
  - Points: `view_user_points(user_id)`, `change_user_points(user_id, delta)`.
  - Discord: `get_user_info(token)` → `/users/@me`.
- **ObjectId handling:** `utils.convert_object_ids()` for JSON-safe IDs.

### 2.4 Auth (`backend/auth.py`)

- **Discord OAuth** (implicit flow). Frontend stores the Discord **access token** in `localStorage` (`discord_token`).
- Backend accepts the token, caches a local session mapping `token → discord_id` (`/api/login_discord`), and verifies via `verify_local_token(token)` or falls back to calling Discord `users/@me`.

> **Prod note:** Issue your own short-lived JWT after initial Discord verification and require it on subsequent calls.

### 2.5 Realtime: Socket.IO

- `socketio_events.py`:
  - On `connect`, binds `sid → user_id` (if token resolves).
  - Tracks `online_users`, emits `online_users_update` on change.
  - Room helpers per user.
- Each game module registers **namespace handlers** under `/crash`, `/coin`, `/mines`.

### 2.6 Games

**Common**
- Games call internal **Points** and **Activity** REST endpoints via `requests.Session()`:
  - `POINTS_API_BASE = "http://localhost:8000/api/points"`
  - `ACTIVITY_API_URL = "http://localhost:8000/api/activity"`
  - Include header `X-API-Key: "test123"` (dev-only).  
- Users initialize to **1000 points** on first touch.
- User identity retrieved from Discord token (`/users/@me`).

> **Security:** Move keys/URLs to env, rotate secrets, and add rate limits.

**Crash** (`backend/games/crash_game.py`)

- **Namespace:** `/crash`
- **Phases:** `BETTING_DURATION = 20s` → Running (multiplier ticks by `CRASH_STEP = 0.01` every `CRASH_INTERVAL = 0.1s`) → Crash (random).  
- **Server:** Tracks bets & target multipliers; settles wins on cashout/target; updates points and logs activity.

**Coin Flip** (`backend/games/coin_game.py`)

- **Namespace:** `/coin`  
- **Flow:** `join_coin` → `place_bet` (side, amount) → server resolves `heads/tails`, applies point delta, logs activity.

**Mines** (`backend/games/mines_game.py`)

- **Namespace:** `/mines`  
- **Board:** 5×5, default 5 mines, `MAX_MINES = 24`.  
- **Flow:** `place_bet` (deduct) → `reveal_cell(r,c)` (lose on mine; else multiplier grows) → `cashout` to lock winnings. Points settle via Points API; activity logged.

### 2.7 REST endpoints (by module)

**Auth**
- `POST /api/login_discord` — cache session for Discord token.

**User & profile** (`endpoints/user.py`)
- `GET /api/user_data`
- `GET /api/get_profile`
- `GET /api/casino_stats`
- `POST /api/buy_item`
- `POST /api/transfer_points`

**Shop** (`endpoints/shop.py`)
- `GET /api/shop_items`

**Secondary shop / marketplace** (`endpoints/secondary_shop.py`)
- `POST /api/resell_spot`
- `GET  /api/get_secondary_shop`
- `POST /api/buy_secondary_item`
- `POST /api/cancel_listing`
- `POST /api/refund_item`

**Spot requests** (`endpoints/request_spot.py`)
- `POST /api/request_spot`
- `GET  /api/get_spot_requests`
- `POST /api/admin_accept_spot`
- `POST /api/admin_deny_spot`

**Admin** (`endpoints/admin.py`)
- `GET  /api/is_admin`
- `POST /api/admin/create_item`

**Activity** (`endpoints/activity.py`)
- `GET  /api/activity`
- `POST /api/activity` (requires `X-API-Key`)

**Vouchers** (`endpoints/voucher.py`)
- `POST /api/redeem_voucher`

**Wallets** (`endpoints/wallet.py`)
- `GET  /api/get_wallets`
- `POST /api/save_wallets`

**Arcade wallet** (`endpoints/arcade_wallet.py`)
- `GET  /api/arcade_wallet/<discord_id>`
- `POST /api/arcade_wallet`

**Points** (`points.py`)
- `GET  /api/points/view?user_id=...`
- `POST /api/points/change` ({{ user_id, change }}, requires `X-API-Key`)

**Deposits** (`endpoints/deposit.py`)
- `POST /deposit` — Solana-style Ed25519 signature verification, then credit points.

### 2.8 Block processor (`backend/block_processor.py`)

- Runs in a daemon thread from `main.py` every `BLOCK_TIME` (default 1s).
- Reads `current_block`; if there are queued “transactions,” applies them, emits socket updates, increments block number.
- Intended as a fairness queue for applying state changes in ticks.

---

## 3) Frontend

### 3.1 Stack

- **Create React App**
- **socket.io-client**
- **Material UI (MUI)**, **Chart.js**
- **axios**
- **@solana/web3.js** for keypair/signature in deposits

Set `REACT_APP_BACKEND` to point to the backend base (defaults to some hard-coded values in a few components).

### 3.2 Auth flow

- `Login.js` builds Discord authorize URL with `response_type=token`, `client_id=1268811331755311104`, scope `identify`, and `redirect_uri=<site>/discord/callback`.
- `DiscordCallback.js` reads `#access_token` from URL fragment and stores `localStorage['discord_token']`.
- REST uses `Authorization: Bearer <token>`; sockets include token during join.

### 3.3 Key components

- `CrashGame.js` — connects to `/crash`, renders chart, handles bets/targets, displays round state.
- `CoinFlip.js` — connects to `/coin`, animates coin, handles bet/outcome.
- `Minesweeper.js` — connects to `/mines`, manages board state and cashout.
- `Deposit.js` — browser Solana keypair; signs message; posts to `/deposit` to credit points.
- `Profile.js` — user profile & balance.
- `Shop.js` — primary shop.
- `Header.jsx` — navigation & auth state.
- `BlockExplorer.js`, `Chat.js`, `RPSGame.js` — utilities/stubs.

---

## 4) Security & fairness

- **Discord tokens:** Consider swapping to **server-issued JWT** (short TTL + refresh) after first Discord verify. Require JWT on all API/socket actions.
- **Secrets:** Move `X-API-Key`, service URLs, and CORS origins to environment. Never hard-code in source.
- **Rate limiting:** Add per-IP and per-user rate limits on bet/place/cashout and `/api/points/change`.
- **RNG:** Prefer `secrets` and/or **provably fair** scheme (server seed commit, post-round reveal, optional client seed). Persist seeds for audits.
- **Deposit replay protection:** Add unique nonce/session per deposit intent to prevent signature reuse.
- **Atomicity:** Route all points changes through a single transactional function to avoid race conditions across games/services.

---

## 5) Running locally

### 5.1 Prereqs
- Python 3.11+, Node 18+, MongoDB.
- Create `backend/auth.env`:
  ```env
  MONGO_URI=mongodb://localhost:27017
  MONGO_DB=arcadeAppDB
  ```

### 5.2 Backend
```bash
cd backend
python -m venv .venv
# Windows: .venv\Scripts\activate
source .venv/bin/activate
pip install -r requirements.txt  # or install: flask flask-cors flask-socketio eventlet pymongo python-dotenv requests pynacl base58
python main.py  # http://localhost:8000
```

### 5.3 Frontend
```bash
cd interface
npm i
# e.g.:
#   export REACT_APP_BACKEND=http://localhost:8000
#   (Windows PowerShell) $env:REACT_APP_BACKEND="http://localhost:8000"
npm start  # http://localhost:3000
```

**Discord OAuth:** Set your application to allow redirect `http://localhost:3000/discord/callback`.

---

## 6) Data model (representative)

**User**
```json
{ "user_id": "discord_id", "username": "Name#1234", "points": 12345, "inventory": [], "roles": [] }
```

**Shop item**
```json
{ "name": "VIP Spot", "image_base64": "...", "price": 500, "spots": 10 }
```

**Secondary listing**
```json
{ "listing_id": "...", "seller_id": "...", "item": { }, "price": 300, "status": "active" }
```

**Voucher store**
```json
{ "codes": { "WELCOME100": 100, "PROMO500": 500 } }
```

**Arcade wallet**
```json
{ "discord_id": "...", "sol_pubkey": "...", "eth": "...", "btc": "..." }
```

**Activity**
```json
{ "user_id": "...", "username": "...", "category": "games", "game_type": "crash|mines|coin_flip",
  "points_bet": 100, "points_profit": 150, "points_earned_total": 250, "multiplier": 2.5, "created_at": "ISO" }
```

---

## 7) Implemented vs. TODO

### Implemented
- Discord implicit auth + local session cache.
- Points service with internal API key.
- Activity log service.
- Primary shop; secondary shop (list/buy/cancel/refund).
- Voucher redemption.
- Wallet CRUD + arcade wallet record.
- Crash, Coin Flip, Mines (Socket.IO) with points settlement & activity.
- Block processor loop for tick-based updates.

### TODO (engineering)
- Harden auth (JWTs), migrate secrets to env, add rate limits.
- Provable fairness (seed commit/reveal), audit logging.
- Centralize/atomize points mutations; add DB indexes.
- Deposit nonces + replay protection.
- Admin CRUD & pagination for listings/activity.
- Tests (pytest + Playwright/Cypress) and CI (GitHub Actions).
- Dockerfiles + Compose; environment-specific configs.

---

## 8) Features to Add (Product & Architecture)

### 8.1 Real SPL Deposits (replace “fake credits”)
**Goal:** Move deposit flow from points credits to **on-chain SPL token deposits** on Solana.

- **UX:** In `Deposit.js`, generate or show a **server-provisioned** deposit address per user (Derive via PDAs or a custodial wallet per user). Display memo/reference.
- **Backend:**
  - Add a **Solana webhook/poller** (e.g., via RPC or webhook provider) to watch for incoming SPL transfers to the user’s deposit address.
  - On confirmed transfer, **credit points 1:1 or per rate**, record tx signature, and mark as finalized (idempotent).
  - Maintain **nonce/reference** to prevent double-counting.
- **Security:** Never trust client signatures for balance changes. Do server-side signature/ownership validation and confirm finality (e.g., 32 confirmations).  
- **Config:** RPC endpoint, mint address, decimals, min deposit.

### 8.2 Staking System (Multiplayer “House”)
**Goal:** Users stake together to form **the house liquidity**; earnings/losses are socialized. This reframes risk as **passive PvP**.

- **Contracts/Off-chain:** Start off-chain with Mongo, then migrate on-chain if desired.
- **Data model:**
  - `pools`: {{ pool_id, total_stake, apy_snapshot?, lock_days }}
  - `stakes`: {{ user_id, pool_id, amount, start_at, unlock_at, harvested }}
  - `pool_pnl_ledger`: aggregated game PnL entries credited/debited to the pool
- **Flows:**
  - **Stake:** Lock for **~30 days** (configurable). Add to `total_stake` and user `stakes`.
  - **PnL accounting:** Each game’s house edge (or net PnL) posts to `pool_pnl_ledger`. On a cadence (e.g., per block or hourly), distribute proportional earnings/losses to stake accounting (not withdrawable until unlock).
  - **Claim:** After lock, users can **claim & withdraw** their principal + share of PnL.
- **APIs:**
  - `POST /api/stake` {{ amount, pool_id }}
  - `GET /api/stakes` (list user stakes, vesting)
  - `POST /api/claim` {{ stake_id }}
  - `GET /api/pool` (pool stats)
- **Fairness/Transparency:** Expose **pool PnL dashboard** and on-chain audit trail if/when migrated.

### 8.3 Block Processor — Randomized Tx Ordering
**Goal:** Maintain a **queue-based fairness** system but avoid “speed advantage”. Instead of strict FIFO, process a **random subset/order** each tick.

- **Design:**
  - At each tick (e.g., every 1s), collect the pending queue `Q`.
  - Compute `K = min(|Q|, MAX_PER_BLOCK)`; choose a **random permutation** of Q and process the first K in that order.
  - Persist the **seed** (e.g., block hash or server-seed + reveal) to allow auditing the selection order for that block.
- **Implementation hints:**
  - Store pending actions with {{ id, user_id, type, payload, enqueued_at }}.
  - On tick: `rng = secrets.SystemRandom()`; `rng.shuffle(Q)`; process.
  - Optionally **weight** by `enqueued_at` buckets to ensure starvation-free fairness.
- **Observability:** Log block number, seed, processed ids, failures; expose `/api/blocks/:n` for debug UI.

---

## 9) Dev “happy path” (manual test)

1. Start Mongo + Backend (8000) + Frontend (3000).
2. Login via Discord → `localStorage['discord_token']` set.
3. Open **Profile**; confirm points.
4. Play **Crash/Coin/Mines**; confirm points settle and **activity** entries appear.
5. Try **Shop** buy & **Secondary Shop** list/buy.
6. Redeem **voucher** and see points increase.
7. Test **Deposit** (current simulated credits).

---

## 10) Ownership & Contacts

- **Owner:** `@fugakux`  
- Add Discord/Email for ops and escalation for handoffs.

---

### Appendix A — Deployment Notes (quick)

- Backend behind a reverse proxy (Caddy/Nginx). Use `eventlet` or `gevent` worker suited for Socket.IO.
- Configure **CORS** to allowed origins only.
- Externalize all secrets & base URLs via env.
- Prefer **Docker** for parity; add `docker-compose.yml` (backend + Mongo + frontend).
