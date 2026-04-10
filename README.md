# PlasticCoin

**Open, permissionless, community-based physical currency — backed by Bitcoin, minted from reclaimed plastic.**

---

## What is PlasticCoin?

PlasticCoin is a protocol for issuing physical coins that carry real Bitcoin value. Each coin is moulded from reclaimed waste plastic, embedded with an NFC chip, and backed 1:1 by Bitcoin held in a community treasury.

When you tap a PlasticCoin with your phone, the browser opens. You see exactly what the coin is worth, whether it's genuine, and whether the Bitcoin backing it is confirmed on the blockchain. No app. No account. No trust required — only verification.

The protocol is designed to start in a single community, prove itself, and replicate. Each new community that adopts it runs its own nodes, holds its own treasury, mints its own coins. Nothing is shared except the standard.

---

## Why

Money was first stamped in gold and silver — scarce, requiring royal assent. Then printed on paper — governments only. Now the substrate is everywhere. Waste plastic has negative value. It is a pollutant, in our bodies and our food.

PlasticCoin reclaims two things simultaneously: the waste plastic itself, and the right to make money. Not by asking permission. By building something honest that works.

The first mint is in a small Italian town of 7,500 residents. Every year the economy gets weaker. The young people leave. The old people die. But the tourists keep coming every summer — spending euros that flow through and leave. PlasticCoin is an attempt to anchor some of that value locally, in an object that belongs to the place.

It starts small. It grows slowly. It proves itself.

---

## How It Works

```
Buyer pays Lightning invoice on Sunmi V2S terminal
        ↓
Sats arrive in merchant Blink Lightning wallet
        ↓
Coin activated in distributed ledger — NDEF URL written to NFC tag
        ↓
Coin circulates physically — tap to verify on any phone
        ↓
Holder returns coin to merchant terminal
        ↓
LNURL QR displayed — holder scans with Lightning wallet
        ↓
Sats flow out of merchant wallet to holder
        ↓
Coin returns to inactive — ready for next cycle
```

Periodically the merchant sweeps their net Lightning balance to the on-chain community treasury. Every coin's backing becomes verifiable on the Bitcoin blockchain.

---

## The Coin

Two parts:

**Body** — 40mm diameter, injection moulded from reclaimed plastic. The unique surface microstructure of the recycled feedstock is enrolled as a Physical Unclonable Function — the coin's physical fingerprint. Body colour varies by feedstock and is part of each community's identity.

**NFC tag** — NTAG213, encased in standardised coloured plastic. The colour is the denomination:

| 🔴 Red | 🔵 Blue | 🟢 Green | ⚪ White | ⚫ Black |
|--------|---------|---------|---------|---------|
| Lowest | · | · | · | Highest |

Denomination values are set once per year on **22 May** by protocol consensus.

---

## The Stack

| Layer | Technology |
|-------|-----------|
| Physical coin | Injection moulded recycled plastic, NTAG213 NFC |
| Verification | NFC (any phone) + ORB optical PUF matching (OpenCV.js) |
| Merchant terminal | Sunmi V2S POS, NFC bridge, capture shoe |
| Lightning payments | Blink wallet API |
| Node software | PlasticCoin Core — Node.js / Express / SQLite |
| Replication | Litestream — real time across 3 Raspberry Pi nodes |
| Treasury | Bitcoin on-chain multi-sig + Lightning |
| Dashboard | Static HTML on Cloudflare Pages |

---

## Repository Structure

```
plasticcoin/
├── PROTOCOL.md                      The protocol specification
├── README.md                        This file
│
├── plasticcoin-core/                Node server
│   ├── server.js                    Main entry point
│   ├── first-run.js                 Interactive mint setup wizard
│   ├── setup.sh                     Raspberry Pi deployment script
│   ├── generate-litestream-config.sh  Three-node replication setup
│   ├── MIGRATION.md                 Merchant webapp migration guide
│   ├── src/
│   │   ├── db/
│   │   │   ├── schema.sql           Database schema v0.2
│   │   │   ├── database.js          Development adapter (sql.js)
│   │   │   └── database.prod.js     Production adapter (better-sqlite3 + WAL)
│   │   ├── middleware/
│   │   │   └── auth.js              X-Bridge-Token authentication
│   │   ├── routes/
│   │   │   ├── coins.js             Mint, redeem, verify, LNURL flow
│   │   │   ├── verify.js            Public coin tap page
│   │   │   ├── sweeps.js            Treasury sweep recording
│   │   │   ├── dashboard.js         Public read-only API
│   │   │   └── mints.js             Mint and merchant registration
│   │   └── services/
│   │       ├── lightning.js         LNURL encoder + Blink payInvoice
│   │       ├── nodes.js             Heartbeat and IP geolocation
│   │       └── reconciliation.js    Automatic 15-minute reconciliation
│   ├── merchant-webapp/
│   │   └── worker.js                Migrated Cloudflare Worker (SatsCASH only)
│   └── test/
│       ├── basic.js                 8 unit tests
│       └── lifecycle.js             34 integration tests — full coin lifecycle
│
└── plasticcoin-dashboard/
    ├── index.html                   Community dashboard (single file)
    └── DEPLOY.md                    Cloudflare Pages deployment guide
```

---

## Quick Start

**Development (any machine with Node.js 18+):**

```bash
cd plasticcoin-core
npm install
cp .env.example .env
# Edit .env with your values
npm start
```

```bash
# Run tests
npm test              # 8 unit tests
npm run test:lifecycle  # 34 integration tests
```

**First mint setup:**

```bash
node first-run.js     # Interactive wizard — registers mint and first merchant
```

**Raspberry Pi deployment:**

```bash
sudo bash setup.sh    # Full automated setup — Node.js, Litestream, systemd services
```

**Dashboard:**

Deploy `plasticcoin-dashboard/index.html` to Cloudflare Pages. Enter your node's IP address in the top-right field. Done.

---

## API

The node exposes a REST API. All coin and ledger operations require `X-Bridge-Token` authentication. Dashboard and verification endpoints are public.

**Core endpoints:**

```
POST /coins/mint          Activate a coin with Bitcoin value
GET  /coins/:uid          Get coin record
POST /coins/:uid/redeem   Redeem coin — returns LNURL for Lightning payment
GET  /verify/:uid         Public coin verification page (opened by NFC tap)
POST /sweeps              Record a Lightning-to-on-chain treasury sweep
GET  /dashboard/summary   Live mint totals and treasury status
GET  /dashboard/merchants Per-merchant sweep status and Lightning balance
GET  /nodes               Node network status and geolocation
GET  /reconcile/latest    Most recent reconciliation result
```

Full API reference in `plasticcoin-core/README.md`.

---

## The Node Network

Each mint runs three Raspberry Pi 4 nodes. One per community keyholder. Each holds a complete copy of the ledger. Litestream replicates changes in real time via SFTP between nodes.

To destroy the ledger an attacker must simultaneously gain physical access to every node across every community in the network. As the protocol grows this becomes progressively harder. No cloud account. No single point of failure. No login to steal.

**Optional: Solar nodes**

Nodes can run entirely on solar power. The reference hardware — Victron MPPT controller, 12v battery, buck converter, 100w flexible panel — costs less than a second-hand laptop and consumes 3-5 watts continuously. A community can place nodes anywhere with sunlight, independent of mains electricity.

---

## Protocol

The full protocol specification is in `PROTOCOL.md`.

Current version: **v0.6**

The protocol defines:
- Physical coin standard (40mm, NTAG213, colour denominations)
- Three-wallet treasury structure
- Dual-layer verification (NFC + PUF optical)
- Distributed ledger requirements
- Lightning payment flow and sweep process
- Reconciliation standard
- Replication requirements for new mints

The protocol does not define denomination values, mint mark design, coin body colour, or community governance. Those belong to each community.

---

## Denomination Standard

Values are set once per year on **22 May**. Each community sets values appropriate to their local economy within the colour framework. The colour order — red lowest, black highest — is fixed across all mints globally.

The first mint launches with low values intentionally. Enough to participate. Not enough to risk. Trust is built slowly.

---

## The First Mint

The first mint is in Italy. A small town. The protocol arrives quietly — as an art project, a green initiative, maybe just interesting jewellery. The coin is beautiful. The plastic is reclaimed. The Bitcoin is real.

The mint mark on the surface is the town's identity pressed into the material. Every coin carries it.

---

## Contributing

The protocol is open. Implementations are welcome. If you are building on this protocol or running a mint, open an issue to register your mint code and join the network.

Areas where contribution is most useful:

- Optical calibration data — similarity threshold values from real prototype testing
- Solar node build documentation — photographs, wiring diagrams, enclosure designs
- Translations — the verification page and dashboard in local languages
- Denomination governance — proposals for the 22 May annual process

---

## Licence

MIT — see `LICENSE`

The protocol itself is in the public domain. No permission required to implement it.

---

## Status

**April 2026 — Pre-launch**

- Protocol: v0.6 draft
- PlasticCoin Core: v0.1.0
- Dashboard: v0.1.0
- First mint: in preparation

The software is tested. The protocol is documented. The first coins are being designed. The first Pi nodes are being assembled.

---

*No central authority. No permission required. The protocol governs it.*
