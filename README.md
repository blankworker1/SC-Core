# SC Core

**Open, permissionless, community-based physical currency — backed by Bitcoin, minted from reclaimed plastic.**

---

## What Is SC Core

SC Core is the node server that runs the PlasticCoin Protocol. It replaces any centralised platform with a locally-owned, distributed SQLite ledger replicated across three Raspberry Pi nodes held by community keyholders.

Each physical coin is a 40mm disc moulded from reclaimed plastic with an embedded NFC chip. When a holder taps the coin on any phone the browser opens to the verification page — no app, no account, no permission required. The page shows denomination, status, and the Bitcoin transaction hash confirming the coin's backing.

---

## Protocol Version

This implementation targets **PlasticCoin Protocol v0.6**

Denomination standard (ascending value):

| Colour | Default |
|--------|---------|
| 🔴 Red | 5,000 sats |
| 🔵 Blue | 10,000 sats |
| 🟢 Green | 50,000 sats |
| ⚪ White | 100,000 sats |
| ⚫ Black | 500,000 sats |

Values set once per year on **22 May** by protocol consensus.

---

## Architecture

```
Coin tap → https://sc.yourdomain.com/verify/:uid
         → Cloudflare Worker (sc-worker — failover router)
         → Tries Node 1 → Node 2 → Node 3 in order
         → First healthy node serves the request
         → SC Core node :3000
         → SQLite ledger (Litestream replication across all nodes)

Merchant terminal (Sunmi V2S)
         → Merchant webapp (worker.js)
         → NFC bridge :8765
         → SC Core node (X-Bridge-Token auth)
         → Blink Lightning wallet (in-memory write key — never stored)

Mint operator
         → Admin panel (sc-admin.html, laptop browser)
         → SC Core node (X-Bridge-Token auth)
         → Treasury Lightning wallet (session-only write key)

Community dashboard
         → Served by Cloudflare Worker from memory (always available)
         → Queries SC Core public API for live data
```

---

## Node Redundancy

Every request to `sc.yourdomain.com` — every coin tap, every dashboard query, every API call — goes through a Cloudflare Worker before reaching any node. The Worker holds a prioritised list of nodes and routes traffic to the first one that responds.

**How failover works:**

1. Worker tries Node 1. If it responds within 3 seconds, the request is proxied there.
2. If Node 1 is unreachable, the Worker immediately tries Node 2.
3. If Node 2 also fails, it tries Node 3.
4. The Worker caches the winning node for 30 seconds so subsequent requests don't re-probe.
5. When the cached node stops responding, the cache expires and the process repeats.

Worst case recovery time is around 33 seconds. After that, all traffic routes automatically to the next live node with no human intervention.

**What this covers:**

- Coin taps (verify page) — fully automatic failover
- Community dashboard — always available, served from Worker memory even if every node is offline
- All public API reads — fully automatic failover
- Mint/redeem writes from the merchant terminal — the terminal points at a specific node URL in its settings. If that node fails, the merchant sees an error and changes the node URL to a different node. This is the one manual step remaining.

**Growing the network:**

Adding a new node to the redundancy pool is a single change — add the node's IP to the `SC_NODES` environment variable in the Cloudflare Worker dashboard. No code change, no redeployment. The new node immediately joins the failover pool.

This is what makes every additional node meaningful. A keyholder who sets up a Pi — fixed location, solar powered, at a community organisation, or mobile at a market stall — extends the network's resilience the moment they join. The community dashboard shows which nodes are online, their response latency, and which one is currently active. The network's health is visible to everyone.

The node map grows as the community grows. Each new node is not just a backup — it is a visible, named, located participant in the network that the community can see and trust.

---

## Requirements

**Development:** Node.js 18+, npm

**Production (each node):** Raspberry Pi 4 2GB, Raspberry Pi OS Lite 64-bit (Bookworm), Node.js LTS, Litestream

---

## Quick Start — Development

```bash
cd plasticcoin-core
npm install
cp .env.example .env
# Edit .env — set BRIDGE_SECRET, NODE_SECRET, NODE_ORIGIN
npm start
```

Test: `curl http://localhost:3000/status`

Run tests: `npm run test:all`

---

## Quick Start — Raspberry Pi

```bash
# On a fresh Raspberry Pi OS Lite installation
sudo bash setup.sh

# After setup — interactive first-run wizard
npm run first-run
```

---

## Configuration

Key `.env` variables:

| Variable | Description |
|----------|-------------|
| `NODE_ID` | Unique node ID e.g. `ITA001-N01` |
| `MINT_CODE` | Community mint code e.g. `ITA001` |
| `MERCHANT_ID` | Default merchant ID |
| `PORT` | API port (default 3000) |
| `NODE_ORIGIN` | Public URL e.g. `https://sc.yourdomain.com` — written into every NFC tag |
| `BRIDGE_SECRET` | Shared with merchant webapp — protects all write operations |
| `NODE_SECRET` | Shared between nodes — protects sync and reconciliation |
| `DB_PATH` | SQLite database file path |
| `PEER_NODES` | Comma-separated URLs of other nodes |
| `TREASURY_BLINK_WALLET_ID` | Treasury Lightning wallet — batch activation payments |
| `TREASURY_BLINK_API_KEY_READONLY` | Treasury read-only key — balance display on dashboard |
| `GEO_ENABLED` | Enable IP geolocation for node map (true/false) |
| `SOLAR_POWERED` | Mark this node as solar powered (true/false) |

**`NODE_ORIGIN` is the most important variable.** It is the permanent subdomain written into every NFC tag at manufacture. Set up a Cloudflare Tunnel pointing `sc.yourdomain.com` to `localhost:3000` before manufacturing any coins. Never change it after coins have been issued.

---

## Security Model

**Write keys are never stored on the server.**

The merchant's Blink write key (needed to pay out Lightning redemptions) is entered by the merchant on the Sunmi terminal and held in the webapp's localStorage — on the terminal only, never transmitted to the node.

The LNURL payment callback receives the write key as a request header (`X-Blink-Api-Key`) passed from the terminal at the moment of redemption. The node processes the payment and the key is gone.

The mint operator's treasury write key is entered in the admin panel at session start, held in a JavaScript object in memory, and cleared when the browser tab closes.

The database stores only read-only Blink keys — for balance display and reconciliation. No write key ever touches disk.

---

## API Reference

### Public — no authentication

| Method | Route | Description |
|--------|-------|-------------|
| GET | `/status` | Node health, version, mint code |
| GET | `/coins/:uid` | Coin record |
| GET | `/verify/:uid` | Coin tap page (NFC opens this) |
| GET | `/coins/lnurlsc/:uid` | LNURL withdraw step 1 |
| GET | `/coins/lnurlsc/callback/:uid` | LNURL withdraw step 2 |
| GET | `/mint/mark` | Mint's registered SVG mark |
| GET | `/dashboard/summary` | Live mint totals, sweep alerts, sleeper batches |
| GET | `/dashboard/coins` | Coin breakdown by denomination |
| GET | `/dashboard/merchants` | Per-merchant sweep status |
| GET | `/dashboard/nodes` | Node map data |
| GET | `/dashboard/activity` | Recent events feed |
| GET | `/dashboard/treasury` | Treasury Lightning wallet balance |
| GET | `/reconcile/latest` | Most recent reconciliation result |
| GET | `/nodes` | Node network status |
| GET | `/mints/:code` | Mint details |

### Admin — requires `X-Bridge-Token`

| Method | Route | Description |
|--------|-------|-------------|
| POST | `/coins/register` | Register NFC UID |
| POST | `/coins/mint` | Mint coin — assign Bitcoin value |
| POST | `/coins/:uid/redeem` | Redeem — generate LNURL |
| POST | `/mints` | Register mint |
| POST | `/mints/merchants` | Register merchant |
| PATCH | `/mints/merchants/:id` | Update merchant |
| GET | `/mints/merchants` | List merchants |
| POST | `/mint/mark` | Upload mintmark SVG |
| POST | `/sweeps` | Record treasury sweep |
| GET | `/sweeps` | Sweep history |
| GET | `/sweeps/status` | Current sweep status per merchant |
| POST | `/sleepers/register` | Register pre-launch sleeper coins |
| GET | `/sleepers` | List sleepers by batch |
| GET | `/sleepers/batches` | Batch summary with activation status |
| GET | `/sleepers/:uid` | Single sleeper record |
| GET | `/sleepers/activation-preview/:batch` | Per-merchant payment breakdown before activation |
| POST | `/sleepers/activate-batch` | Launch day — activate all sleepers in batch |

### Node — requires `X-Node-Token`

| Method | Route | Description |
|--------|-------|-------------|
| POST | `/nodes/heartbeat` | Node presence update |
| POST | `/nodes/sync` | Ledger state for reconciliation |
| POST | `/reconcile` | Run reconciliation immediately |

---

## Roles

**Merchant** — operates the Sunmi V2S terminal. Mints and redeems coins. Holds their own Blink write key on the terminal — never shared with the node. From the merchant's perspective every coin is either ready or not. No admin functions.

**Mint operator** — holds node hardware, treasury wallets, admin token. Registers sleeper coins, uploads the mintmark, triggers batch activation on launch day, monitors treasury balance. Uses `sc-admin.html` on a laptop — not the Sunmi terminal.

---

## Pre-Launch Sleeper Coins

A sleeper coin is registered in the ledger — UID, denomination colour, batch — before it carries any Bitcoin value. When a holder taps a sleeper coin the browser opens to a completely blank white page. No logo, no message. The coin appears inert.

On launch day the mint operator opens the admin panel, previews the per-merchant payment breakdown, manually transfers sats from the treasury Lightning wallet to each merchant's Blink wallet, then triggers batch activation. Every sleeper in the batch simultaneously becomes active: denomination value assigned, NDEF URL confirmed, status flipped to active. The next tap returns the full verification page.

The verify count increments silently during the sleeper period — the operator can see how many times each coin has been tapped before launch.

**Sleeper workflow:**

```
Coins manufactured
  → operator registers UIDs via admin panel (POST /sleepers/register)
  → coins sold physically by merchant — ledger records merchant assignment

Before launch:
  → operator funds treasury Lightning wallet (single-sig → Lightning transfer)
  → admin panel shows per-merchant payment amounts (GET /sleepers/activation-preview/:batch)
  → operator manually pays each merchant wallet via Blink

Launch day:
  → operator enters on-chain tx hash, checks confirmations
  → clicks Activate Batch in admin panel (POST /sleepers/activate-batch)
  → all coins wake up simultaneously
  → every subsequent tap returns the full verification page
```

---

## Mintmark

Each mint registers a single SVG mark — the visual identity pressed into every coin and displayed across all interfaces.

The mark is always rendered as black on a white framed square (`background: #ffffff`, `border-radius: 6px`, `1px solid rgba(0,0,0,0.15)`). Context-independent — works on the dark terminal, light dashboard, verify page, printed receipt, or pressed into plastic.

Upload via the admin panel Mintmark section. Black fills are automatically normalised to `currentColor` on ingest. The mark appears in:

- The merchant webapp top bar (34px, right slot)
- The coin verification page (32px, beside the SC logo)
- The community dashboard header (32px)

**Endpoint:** `GET /mint/mark` — public, cached 24h, returns raw SVG.

---

## Merchant Webapp

`merchant-webapp/worker.js` runs on the Sunmi V2S terminal as a Cloudflare Worker.

**Three-zone header:** back button (left, context-sensitive) — SC logo (center) — mintmark (right).

**Settings screen configuration:**
- Node URL — points the terminal at the SC Core node
- Store name — displayed in receipts
- Blink wallet ID — the merchant's Lightning wallet
- Blink API key (write) — held in localStorage on the terminal only

When the operator saves the node URL, the webapp silently fetches the mintmark from `NODE_ORIGIN/mint/mark` and stores it locally. All headers update automatically.

**Migration:** `MIGRATION.md` covers the transition from the original Cloudflare Worker.

---

## Cloudflare Tunnel Setup

```bash
# On the Raspberry Pi
curl -L -o cloudflared.deb \
  https://github.com/cloudflare/cloudflared/releases/latest/download/cloudflared-linux-arm64.deb
sudo dpkg -i cloudflared.deb

cloudflared tunnel login
cloudflared tunnel create sc-verify
```

Config at `/home/plasticcoin/.cloudflared/config.yml`:

```yaml
tunnel: YOUR-TUNNEL-UUID
credentials-file: /home/plasticcoin/.cloudflared/YOUR-TUNNEL-UUID.json
ingress:
  - hostname: sc.yourdomain.com
    service: http://localhost:3000
  - service: http_status:404
```

```bash
cloudflared tunnel route dns sc-verify sc.yourdomain.com
sudo cloudflared service install
sudo systemctl enable cloudflared
sudo systemctl start cloudflared
```

Test: `curl https://sc.yourdomain.com/status`

Set `NODE_ORIGIN=https://sc.yourdomain.com` in `.env`.

---

## Three-Node Replication

```bash
# On each Pi after setup.sh
sudo bash generate-litestream-config.sh

# Exchange SSH keys with peer nodes
ssh-copy-id -i /home/plasticcoin/.ssh/id_rsa.pub plasticcoin@NODE2_IP
ssh-copy-id -i /home/plasticcoin/.ssh/id_rsa.pub plasticcoin@NODE3_IP

sudo systemctl restart litestream
```

---

## Deployment Sequence

1. Register domain, set up Cloudflare DNS
2. `sudo bash setup.sh` on each of 3 Pis
3. Configure Cloudflare Tunnel on Node 1 — confirm `sc.yourdomain.com/status` responds
4. `npm run first-run` on Node 1
5. `sudo bash generate-litestream-config.sh` on each Pi, exchange SSH keys
6. Set `NODE_ORIGIN=https://sc.yourdomain.com` in `.env`, restart service
7. Deploy the Cloudflare Worker (`sc-worker/worker-combined.js`) to `sc.yourdomain.com`
   — set `SC_NODES` with all three node IPs, `SC_MINT_CODE`, and timeout values
   — confirm `https://sc.yourdomain.com/_failover/status` returns node health
   — confirm `https://sc.yourdomain.com/` serves the community dashboard
8. Open admin panel (`sc-admin.html`) — upload mintmark
9. Set Node URL in merchant webapp settings — mintmark appears automatically
10. Manufacture first coins — register UIDs as sleepers or mint directly

When adding a new node later: add its IP to `SC_NODES` in the Worker environment.
No redeployment of SC Core required. The node joins the redundancy pool immediately.

---

## File Structure

```
plasticcoin-core/
├── server.js                         Main entry point
├── first-run.js                      Interactive mint setup wizard
├── setup.sh                          Raspberry Pi deployment script
├── generate-litestream-config.sh     Three-node replication setup
├── MIGRATION.md                      Merchant webapp migration guide
├── .env.example                      Configuration reference
├── src/
│   ├── db/
│   │   ├── schema.sql                Database schema v0.2
│   │   ├── database.js               Development adapter (sql.js)
│   │   └── database.prod.js          Production adapter (better-sqlite3 + WAL)
│   ├── middleware/
│   │   └── auth.js                   X-Bridge-Token / X-Node-Token authentication
│   ├── routes/
│   │   ├── coins.js                  Mint, redeem, verify, LNURL flow
│   │   ├── verify.js                 Public coin tap page
│   │   ├── sweeps.js                 Treasury sweep recording
│   │   ├── dashboard.js              Public read-only API + treasury balance
│   │   ├── mints.js                  Mint registration, merchant management, mintmark
│   │   └── sleepers.js               Pre-launch sleeper coin system
│   └── services/
│       ├── lightning.js              LNURL encoder, Blink payInvoice
│       ├── nodes.js                  Node heartbeat, IP geolocation
│       └── reconciliation.js         Automatic 15-minute reconciliation
├── merchant-webapp/
│   ├── worker.js                     Merchant terminal webapp
│   └── worker-original.js            Original unmodified worker
└── test/
    ├── basic.js                      8 unit tests
    ├── lifecycle.js                  34 integration tests
    └── sleeper.js                    20 sleeper system tests
```

**Separate repositories / directories:**

```
sc-worker/
├── worker-combined.js    Combined Cloudflare Worker (dashboard + failover router)
├── worker.js             Base failover logic (source — rebuilt into combined by build.sh)
├── build.sh              Embeds updated dashboard into worker-combined.js
├── wrangler.toml         Cloudflare CLI deployment config
└── SETUP.md              Deployment guide

sc-admin.html             Mint operator admin control panel (single file, open in browser)
satscash-dashboard.html   Community dashboard (embedded in Worker, also deployable standalone)
```

---

## Tests

```bash
npm test                 # 8 unit tests
npm run test:lifecycle   # 34 integration tests — full coin lifecycle
npm run test:sleeper     # 20 sleeper tests — pre-launch system
npm run test:all         # all 62 tests
```

---

## Admin Panel

`sc-admin.html` — open in a browser on the operator's laptop. Not served by SC Core. No installation.

Session credentials (node URL, admin token, treasury wallet ID, treasury write key) are entered at startup, held in a JavaScript object in memory, and cleared when the tab closes. Nothing persists.

Panels: Overview · Treasury · Sleeper Coins · Activate Batch · Merchants · Mintmark · Node Status

---

## Licence

MIT
