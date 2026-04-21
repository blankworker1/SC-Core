# PlasticCoin Core — SatsCASH v3

Bitcoin physical tokens backed by Lightning. A distributed ledger running on community-held Raspberry Pi nodes with no single point of control.

**Protocol version:** v3.0 (bucket model)
**Implementation version:** v3.0.0
**Migration from:** PlasticCoin Core v0.2 (fixed denomination model)

---

## What This Is

PlasticCoin Core is the backend that runs on each community node. It manages a ledger of NFC coins, each of which is a physical bearer instrument backed by real sats held in a three-layer treasury. The system is designed so that no single person can drain the treasury, no single node failure takes the system down, and the reserve backing is publicly verifiable by anyone.

**No third-party platform holds the ledger.** Three Raspberry Pi nodes held by community keyholders each carry a full SQLite copy, replicated continuously via Litestream over SSH. Any one node can serve all operations. The other two are live standbys.

**Write keys are never stored on the server.** The Blink Lightning API write key — needed to pay outgoing redemptions — lives only on the merchant's Sunmi terminal in localStorage. It is sent as a header at the moment of redemption and held in a 10-minute in-memory session. Nothing touches disk. A compromised node cannot drain any wallet.

---

## Architecture

```
  Sunmi V2S Terminal
  ├── Merchant Webapp (Cloudflare Worker)
  └── NFC Bridge APK (localhost:8765)
           │  X-Bridge-Token auth  │  X-Blink-Api-Key at redemption only
           ▼                       ▼
  ┌────────────────────────────────────────────────────┐
  │           PlasticCoin Core Node (Primary)          │
  │           Express API  :3000                       │
  │           SQLite + WAL mode                        │
  │           Litestream replication                   │
  └──────────────────┬─────────────────────────────────┘
           SFTP replication (continuous)
           ▼                       ▼
  ┌─────────────┐         ┌─────────────┐
  │   Node 2    │         │   Node 3    │
  │  RPi 4/5    │◄───────►│  RPi 4/5   │
  └─────────────┘         └─────────────┘

  Cloudflare Worker (sc-worker)
  ├── Dashboard HTML — always available, served from Worker memory
  ├── Failover router — tries Node 1 → Node 2 → Node 3
  └── /coin/:uid — public holder verify page (proxies to node)

  sc-admin.html — standalone file, open in operator's laptop browser
  No installation. Session credentials in memory only.
```

---

## The Coin Model

Each physical coin is a bearer instrument. Whoever holds it owns the value. The coin has no expiry and no registered owner.

**Bucket model.** Each coin has a `cap_sats` ceiling (set annually on Pizza Day, 22 May) and a `balance_sats` that fills and drains freely between zero and cap. The coin is never destroyed — it circulates indefinitely, loaded and redeemed repeatedly.

**Merchant credit system.** Merchants fund a credit line by sending sats to the Mint treasury. They can load coins up to their available credit. When a coin is redeemed, the credit is restored. This keeps merchants accountable without recording who pays to load any individual coin. Anyone with a Lightning wallet can fund a load — the system is anonymous.

**Three-wallet treasury.** Reserves are held across three layers:

| Layer | Type | Role |
|---|---|---|
| Blink Lightning wallet | Hot, live | Operational float — funds redemptions immediately |
| Single-sig on-chain | Warm, declared | Operational reserve — backs larger issuance |
| Multisig cold storage | Cold, declared | Strategic reserve — community governance |

Total reserves are publicly verifiable. On-chain addresses are published and anyone can check mempool.space independently. The reserve ratio is displayed on the public dashboard at all times.

---

## Coin Lifecycle

```
Manufacture → Enrol (NFC UID registered, optical PUF captured)
           → Empty (coin exists, no balance)
           → Active (merchant loads sats via Lightning invoice)
           → Empty (holder redeems via LNURL-withdraw)
           → Active (reloaded again — repeats indefinitely)
```

---

## Requirements

**Development (laptop):**
- Node.js LTS 18+
- npm

**Production (each node):**
- Raspberry Pi 4 or Pi 5
- Raspberry Pi OS Lite 64-bit (Bookworm)
- Node.js LTS
- Litestream

---

## Quick Start — Development

```bash
git clone <repo-url>
cd plasticcoin-core
npm install
cp .env.example .env
# Edit .env with your values
npm start
```

Server starts at `http://localhost:3000`. Test it:

```bash
curl http://localhost:3000/status
curl http://localhost:3000/dashboard/summary
```

Open `sc-admin.html` in your browser, connect to `http://localhost:3000` with your bridge token, and run through the setup form to configure the mint.

---

## Quick Start — Raspberry Pi

```bash
# On a fresh Raspberry Pi OS Lite installation
# Copy the project to the Pi, then:
sudo bash setup.sh
```

The setup script handles everything: Node.js, Litestream, the `plasticcoin` system user, `.env` configuration, two systemd services, log rotation, SSH keys for replication, and UFW firewall rules (port 3000 blocked from public internet).

---

## Migration from v0.2

If you have an existing v0.2 database (fixed denomination model), run the migration before starting the server:

```bash
# Always back up first
cp /path/to/plasticcoin.db /path/to/plasticcoin.db.backup-$(date +%Y%m%d)

# Run the migration
sqlite3 /path/to/plasticcoin.db < migrate-v3.sql

# Verify
sqlite3 /path/to/plasticcoin.db "SELECT * FROM schema_versions;"
sqlite3 /path/to/plasticcoin.db "SELECT * FROM mint_totals;"

# Drop backup table after confirming migration succeeded
sqlite3 /path/to/plasticcoin.db "DROP TABLE IF EXISTS coins_v2;"
```

The migration preserves all existing coin records. Active coins carry their sats value as `balance_sats`. Inactive coins start at zero balance with the new cap applied.

---

## Deployment Sequence (First Node)

1. Register domain, set up Cloudflare DNS
2. `sudo bash setup.sh` on each of three Pis
3. Configure Cloudflare Tunnel on Node 1 — confirm `sc.yourdomain.com/status` responds
4. `npm run first-run` on Node 1 (interactive mint setup wizard)
5. `sudo bash generate-litestream-config.sh` on each Pi — exchange SSH keys between nodes
6. Set `NODE_ORIGIN=https://sc.yourdomain.com` in `.env`, restart service
7. Open `sc-admin.html` — connect to node, complete setup form
8. Set `NODE_URL=https://sc.yourdomain.com` in the Cloudflare Worker environment
9. Deploy `merchant-webapp/worker.js` to Cloudflare Workers
10. On the Sunmi: open `https://your-worker.workers.dev/app/settings`, register terminal
11. Manufacture first coins — enrol UIDs via the Enrol screen

---

## Environment Variables

```bash
# Node identity
NODE_ID=ITA001-N01          # Unique ID for this node
MINT_CODE=ITA001            # Mint code — same on all nodes
NODE_ORIGIN=https://sc.yourdomain.com   # Public URL for this node

# Authentication
BRIDGE_SECRET=              # X-Bridge-Token — shared with merchant terminals
NODE_SECRET=                # X-Node-Token — shared between nodes

# Blink Lightning treasury (read key for balance display)
BLINK_API_KEY=              # Read-only API key
BLINK_API_KEY_READONLY=     # Read-only key (same as above or separate)
BLINK_WALLET_ID=            # Treasury wallet ID

# Blink webhook (for payment settlement notifications)
BLINK_WEBHOOK_SECRET=       # HMAC secret from Blink dashboard

# Database
DB_PATH=./data/plasticcoin.db
PORT=3000
```

**Never stored in the database or on disk:**
- Blink write API key — lives only in merchant terminal localStorage, sent as `X-Blink-Api-Key` header at redemption time
- Admin treasury write key — entered in sc-admin.html session memory, cleared on tab close

---

## API Endpoints

### Coins (merchant terminal)

| Method | Route | Auth | Description |
|---|---|---|---|
| POST | /coins/enrol | Admin | Register new coin UID + store optical descriptors |
| GET | /coins/:uid | Public | Coin status and balance |
| POST | /coins/:uid/load | Admin | Generate invoice to load sats |
| GET | /coins/:uid/load/:hash/status | Admin | Poll load invoice settlement |
| POST | /coins/:uid/redeem | Admin | Generate LNURL-withdraw |
| GET | /coins/:uid/optical | Admin | Retrieve optical descriptors |
| GET | /coins/list | Admin | List all coins for this mint |

### LNURL (called by holder's Lightning wallet)

| Method | Route | Auth | Description |
|---|---|---|---|
| GET | /lnurlsc/:uid | Public | LNURL-withdraw step 1 — parameters |
| GET | /lnurlsc/callback/:uid | Public | LNURL-withdraw step 2 — pay invoice |

### Merchant credit

| Method | Route | Auth | Description |
|---|---|---|---|
| POST | /merchant/register | Admin | Register terminal — returns merchant_id |
| GET | /merchant/:id | Admin | Credit position and today's activity |
| POST | /merchant/:id/sync | Admin | Refresh wallet balance from Blink |
| POST | /merchant/:id/topup | Admin | Generate top-up invoice |
| GET | /merchant/:id/topup/:hash/status | Admin | Poll top-up settlement |

### Admin (Mint operator)

| Method | Route | Auth | Description |
|---|---|---|---|
| POST | /admin/setup | Admin | First-run or update mint config |
| GET | /admin/status | Admin | Full mint config and live totals |
| POST | /admin/reserves | Admin | Update declared on-chain balances |
| POST | /admin/cap | Admin | Update annual coin cap (Pizza Day) |
| GET | /admin/merchants | Admin | All merchants with credit positions |
| PATCH | /admin/merchants/:id | Admin | Suspend or reactivate a terminal |

### Dashboard (public, read-only)

| Method | Route | Auth | Description |
|---|---|---|---|
| GET | /dashboard/summary | Public | Mint totals, treasury, solvency |
| GET | /dashboard/merchants | Public | Per-merchant credit breakdown |
| GET | /dashboard/nodes | Public | Node health and map data |
| GET | /dashboard/activity | Public | Recent events feed |
| GET | /dashboard/coins | Public | Coin breakdown by status |
| GET | /dashboard/treasury | Public | Three-layer treasury with verify links |

### Webhook

| Method | Route | Auth | Description |
|---|---|---|---|
| POST | /webhook/blink | HMAC | Blink payment settlement notifications |

### Existing (unchanged)

| Method | Route | Auth | Description |
|---|---|---|---|
| POST | /sweeps | Admin | Record a treasury sweep |
| GET | /sweeps | Admin | Sweep history |
| POST | /mints | Admin | Register mint (first run) |
| GET | /mints/:code | Public | Mint details |
| GET | /verify/:uid | Public | Public coin tap page (HTML) |
| POST | /nodes/heartbeat | Node | Node health report |
| GET | /status | Public | Node status |

**Auth types:**
- `Admin` — `X-Bridge-Token: <BRIDGE_SECRET>` header
- `Node` — `X-Node-Token: <NODE_SECRET>` header
- `HMAC` — `Blink-Signature` header (SHA-256 of request body)
- `Public` — no authentication

---

## Three-Node Replication

```bash
# On each Pi after setup.sh
sudo bash generate-litestream-config.sh

# Exchange SSH keys with peer nodes
ssh-copy-id -i /home/plasticcoin/.ssh/id_rsa.pub plasticcoin@NODE2_IP
ssh-copy-id -i /home/plasticcoin/.ssh/id_rsa.pub plasticcoin@NODE3_IP

sudo systemctl restart litestream
sudo systemctl status litestream
```

Each node continuously replicates its SQLite WAL to the other two over SFTP. If the primary node fails, the Cloudflare Worker failover router automatically routes to the next healthy node. No manual intervention required.

---

## Service Management

```bash
# Check status
sudo systemctl status plasticcoin
sudo systemctl status litestream

# View logs
sudo journalctl -u plasticcoin -f
sudo journalctl -u litestream -f

# Restart
sudo systemctl restart plasticcoin

# View last 50 log lines
sudo journalctl -u plasticcoin -n 50 --no-pager
```

---

## The Merchant Terminal

The merchant webapp (`merchant-webapp/worker.js`) is a Cloudflare Worker that serves the Sunmi V2S terminal UI. It is entirely separate from the Core node and has no server-side state. All data operations go directly from the terminal to the Core node via the NFC Bridge APK.

**Screens:**
- **Hub** — four action buttons, live credit bar, top-up warning
- **Load Coin** — tap coin, enter amount, display Blink invoice QR, confirm on settlement
- **Redeem Coin** — tap coin, confirm amount, display LNURL QR, confirm on balance drop
- **Verify Coin** — tap coin, show balance and capacity
- **Enrol Coin** — tap new coin, write NDEF, register with node
- **Settings** — terminal config, merchant registration, credit top-up flow

**Bridge APK** (`localhost:8765`) handles:
- `POST /nfc/poll` — waits for NFC tap, returns UID
- `POST /nfc/write` — writes NDEF URL to coin
- `POST /print` — sends receipt to Sunmi thermal printer

Receipt printing fires automatically after successful load (optional, user-initiated) and after successful redeem (automatic). The Blink write key is stored in terminal localStorage only and sent as `X-Blink-Api-Key` at redemption time. It is never transmitted to or stored by the Core node.

---

## The Admin Panel

`sc-admin.html` is a standalone HTML file. Open it directly in any browser on the operator's laptop. No installation, no server.

On connect it prompts for the node URL and admin token. These are held in a plain JavaScript object for the duration of the browser session and discarded when the tab closes. Nothing is written to disk.

**Four tabs:**
- **Overview** — solvency badge, six stat cards, recent activity feed
- **Treasury** — three-layer display with live Lightning balance, declared on-chain balances, mempool.space verify links, and a form to update declared balances
- **Merchants** — table of all registered terminals with credit positions, top-up warnings, and suspend/activate controls
- **Setup** — full mint configuration form

---

## Security Model

**Write keys never touch the server.** The Blink Lightning write key is the only credential that can move funds. It lives on the merchant terminal in localStorage, is sent as a request header only at the moment of redemption, held in a 10-minute in-memory Map on the node, and cleared immediately after the invoice is paid. Nothing reaches disk.

**Timing-safe authentication.** The `BRIDGE_SECRET` and `NODE_SECRET` tokens are compared using `crypto.timingSafeEqual` in all auth middleware, preventing timing attacks.

**Firewall.** `setup.sh` configures UFW to block port 3000 from the public internet. Core is only reachable from localhost, LAN, and via Cloudflare Tunnel.

**Three-node distribution.** No single keyholder controls the ledger. The three nodes are held by independent community members. Litestream replication ensures all three stay in sync. The Cloudflare failover router means no planned maintenance or node failure causes downtime.

**Reserve guard.** The `POST /admin/reserves` endpoint refuses any update where the new declared total would fall below current circulation. It is architecturally impossible to accidentally make the system insolvent via an admin update.

**Anonymous loads.** The system deliberately does not record who pays to load a coin. The merchant credit system enforces accountability — a merchant cannot load more than they have funded — without requiring payer identity. The coin is a bearer instrument.

---

## Annual Maintenance — Pizza Day (22 May)

Each year on Bitcoin Pizza Day the Mint operator reviews and updates the annual coin cap. The cap controls the maximum balance any single coin can hold.

```bash
# Via sc-admin.html → Treasury tab → Annual Coin Cap
# Or directly:
curl -X POST https://sc.yourdomain.com/admin/cap \
  -H "X-Bridge-Token: <BRIDGE_SECRET>" \
  -H "Content-Type: application/json" \
  -d '{"coin_cap_sats": 1000000, "cap_year": 2027}'
```

Existing coins retain their enrolled cap. Only newly enrolled coins after the update use the new cap. This preserves predictability for existing holders.

---

## File Structure

```
plasticcoin-core/
├── server.js                         Main entry point — Express app, route mounting
├── first-run.js                      Interactive mint setup wizard
├── setup.sh                          Raspberry Pi deployment script (11 steps)
├── generate-litestream-config.sh     Three-node replication setup
├── migrate-v3.sql                    Schema migration from v0.2 to v3.0
├── MIGRATION.md                      Step-by-step migration guide
├── .env.example                      Configuration reference
├── src/
│   ├── db/
│   │   ├── schema.sql                Database schema (reference — migration manages updates)
│   │   ├── database.js               Development adapter (sql.js, in-memory)
│   │   └── database.prod.js          Production adapter (better-sqlite3, WAL mode)
│   ├── middleware/
│   │   └── auth.js                   X-Bridge-Token / X-Node-Token (timing-safe)
│   ├── routes/
│   │   ├── coins.js                  Enrol, load, redeem, LNURL flow, optical
│   │   ├── credits.js                Merchant registration and credit top-up
│   │   ├── webhook.js                Blink payment settlement handler
│   │   ├── admin.js                  Mint config, treasury, cap, merchant management
│   │   ├── dashboard.js              Public read-only API — summary, treasury, activity
│   │   ├── verify.js                 Public coin tap page (HTML)
│   │   ├── sweeps.js                 Treasury sweep recording
│   │   ├── mints.js                  Mint registration, merchant CRUD, mintmark
│   │   └── sleepers.js               Pre-launch sleeper coin system
│   └── services/
│       ├── lightning.js              LNURL encoder, Blink payInvoice, createInvoice
│       ├── nodes.js                  Node heartbeat, IP geolocation
│       └── reconciliation.js         Automatic 15-minute reconciliation
├── merchant-webapp/
│   └── worker.js                     Sunmi terminal webapp (Cloudflare Worker)
├── sc-admin.html                     Standalone Mint admin panel (open in browser)
└── test/
    ├── basic.js                      8 unit tests
    ├── lifecycle.js                  34 integration tests
    └── sleeper.js                    20 sleeper system tests
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

## Cloudflare Worker Environment Variables

The merchant webapp Worker requires one environment variable set in the Cloudflare dashboard:

```
NODE_URL = https://sc.yourdomain.com
```

Set via: Cloudflare Dashboard → Workers → your-worker → Settings → Variables.

---

## Protocol

The coin model, treasury rules, and annual cap convention are documented in `PROTOCOL.md`.

Protocol version: **v3.0** (bucket model — replaces fixed denomination v0.6)

**Key protocol changes from v0.6:**
- Fixed denominations (red/blue/green/white/black) replaced by variable bucket per coin
- Annual cap replaces fixed denomination value
- Merchant credit line replaces sweep-based accountability
- Three-wallet treasury replaces single Blink wallet
- Anonymous load added — anyone with a Lightning wallet can fund a coin

---

## Licence

MIT
