# PlasticCoin Protocol

**Version:** v3.0
**Date:** May 2026
**Status:** Active

---

## Overview

PlasticCoin is an open protocol for community-held Bitcoin bearer instruments. A physical coin circulates as cash. Its value is held in Lightning. Anyone can verify it by tapping it to a phone. No account. No app. No identity required.

The protocol defines the physical coin standard, the ledger and treasury requirements, the verification flow, and the governance conventions that allow multiple independent mints to interoperate. It does not define denomination values, mint mark design, or community governance — those belong to each community.

---

## Version History

| Version | Date | Key change |
|---|---|---|
| v0.1 | Oct 2024 | Initial fixed-denomination model |
| v0.4 | Jan 2025 | NFC + optical PUF dual-layer verification |
| v0.6 | Mar 2025 | Three-node replication, sweep model |
| **v3.0** | **May 2026** | **Bucket model replaces fixed denominations** |

### What Changed in v3.0

The fixed-denomination model (red/blue/green/white/black coins with preset sat values) has been replaced by a **bucket model**. Each coin has a `cap_sats` ceiling and a `balance_sats` that fills and drains freely. This change was made for three reasons:

First, fixed denominations aged poorly as Bitcoin appreciated. A black coin worth €85 one year was worth €170 the next. The community had to re-mint to adjust values.

Second, fixed denominations limited flexibility. A holder wanting €40 had to choose between a €17 green and a €42 white. The bucket model lets any amount be loaded onto any coin.

Third, the credit system that replaced the sweep model requires variable amounts to function correctly. A merchant's credit line is drawn against by the exact amount loaded, not by a denomination step.

The colour framework is retired. Coins are now physically distinguished by the community's design choices — mintmark, material, shape — not by denomination colour.

---

## The Coin

A SatsCASH coin is a two-part object: a plastic body and an NFC tag. Together they form a bearer instrument. The value is not on the object — it is in the ledger, backed by sats in the treasury. The object is the key.

### Physical Specification

**Body**
- Outer diameter: 43mm (determined by O-ring specification)
- Plastic core: 34.8mm diameter, ~6mm thick
- Material: injection-moulded recycled plastic (Smile Materials Blue Denim or equivalent)
- Retention: 35mm internal diameter silicone O-ring, 4mm cross-section
- The O-ring holds the plastic core securely and is the visible outer ring of the coin

**NFC Tag**
- Type: NTAG213 or NTAG215 (ISO 14443 Type A, NDEF capable)
- Form factor: 14mm diameter × 2.5mm thick injection-mouldable tag
- Position: rear face of coin, centred, recessed 2.5mm deep
- Written data: single NDEF URI pointing to `{node_origin}/coin/{uid}`

**Optical PUF**
- The surface microstructure of the recycled plastic feedstock is enrolled as a Physical Unclonable Function
- Captured at manufacture using the standardised capture shoe (70mm standoff, flash-only, no ambient light)
- ORB feature descriptors stored in the node database at enrolment
- Used as a secondary verification layer alongside NFC

**Mint Mark**
- Each mint presses its own mark into the coin surface
- Design, placement, and artwork belong to the community
- The mintmark is stored as SVG in the node database and served from `/mint/mark`

### Coin Identity

A coin's identity is its NFC UID — the factory-set, read-only unique identifier of the NTAG chip. It is 7 bytes, globally unique, and cannot be cloned or changed. This UID is the primary key for all ledger records.

There is no serial number on the physical coin. The UID is the serial number. Tapping the coin to any NFC phone reveals it.

---

## The Bucket Model

### Cap and Balance

Every coin has two values in the ledger:

`cap_sats` — the maximum balance the coin can hold. Set at enrolment from the mint's current annual cap. Immutable for the life of that coin.

`balance_sats` — the current balance. Starts at zero. Fills when sats are loaded. Drains when sats are redeemed. Constrained to `0 ≤ balance_sats ≤ cap_sats` at all times.

### Annual Cap

The coin cap is set once per year by the mint operator, by convention on **Bitcoin Pizza Day (22 May)**. The cap is denominated in sats and set to reflect a useful fiat value at current exchange rates — enough to matter, not enough to represent significant financial risk.

Existing coins retain their enrolled cap indefinitely. Only newly enrolled coins after a cap update use the new cap. This preserves predictability for holders and merchants who know what a given coin's ceiling is.

Example: if the cap is set at 1,000,000 sats (≈€900 at current rates) and Bitcoin doubles, the cap the following Pizza Day may be raised or left unchanged. It may not be reduced.

**The cap can only ever increase.** This is enforced at the protocol software level — the node will reject any cap update where the proposed value is less than the current cap, or where the cap year is earlier than the current cap year. This is a hard constraint, not a warning.

The rationale is holder protection. A coin enrolled at a cap of 1,000,000 sats must always be reloadable up to 1,000,000 sats. Reducing the mint cap after enrolment would break that guarantee. Existing coins always retain their enrolled cap — only newly enrolled coins use the updated cap.

### Coin Status

| Status | Meaning |
|---|---|
| `empty` | Enrolled, balance is zero |
| `active` | Balance > 0, ready to circulate |
| `redeeming` | Redemption in progress (LNURL issued, awaiting payment) |
| `suspended` | Suspended by mint operator |

A coin moves between `empty` and `active` indefinitely. There is no retired or expired status in v3.0. A coin circulates for as long as the physical object survives.

---

## The Three-Wallet Treasury

All sats backing circulating coins are held in one of three treasury layers. The total across all three layers must always be greater than or equal to the total balance of all active coins — this is the solvency requirement.

### Layer 1 — Lightning Wallet (Operational Float)

A Blink Lightning wallet holds enough sats to fund day-to-day redemptions. This balance is fetched live from the Blink API and displayed on the public dashboard. It is the only layer with a live balance feed.

Purpose: immediate liquidity for redemptions. Not intended to hold the full reserve.

### Layer 2 — Single-Sig On-Chain (Operational Reserve)

A standard single-signature Bitcoin wallet. Balance is declared by the mint operator and updated manually after on-chain verification. The on-chain address is published so anyone can verify the declared balance independently on a block explorer.

Purpose: larger reserve that backs significant issuance. Can be swept to the Lightning wallet as needed to fund redemptions.

### Layer 3 — Multisig Cold Storage (Strategic Reserve)

A multi-signature Bitcoin wallet requiring multiple keyholders to sign. Balance is declared by the mint operator. On-chain address is published for independent verification.

Purpose: long-term reserve. Provides confidence that the system remains solvent even if the operational layers are depleted. Governance of this wallet belongs to the community — not to any single operator.

### Solvency Rule

```
total_reserves ≥ total_active_balance
```

Where:
- `total_reserves` = Lightning balance + singlesig declared balance + multisig declared balance
- `total_active_balance` = sum of `balance_sats` across all coins with status `active`

This is enforced by the node software on every reserve update. An update that would make the system insolvent is rejected. The ratio is displayed publicly on the dashboard at all times.

---

## The Merchant Credit System

### Purpose

Merchants are the operational backbone of the system. They enrol coins, run load and redeem transactions, and hold Sunmi terminals. The credit system is how merchant accountability is enforced without requiring identity records or transaction surveillance.

### Mechanics

A merchant funds a credit line by sending sats to the Mint treasury. This creates a credit line equal to the amount funded.

When a merchant loads a coin, their `credit_available_sats` decreases by the load amount and their `credit_deployed_sats` increases by the same amount.

When a coin is redeemed at that merchant's terminal, the credit is restored: `credit_available_sats` increases, `credit_deployed_sats` decreases.

A merchant cannot load more coins than their available credit. This is enforced by the node — the load request is rejected if `amount_sats > credit_available_sats`.

### Why This Works

The credit system keeps merchants honest without recording who pays for anything:

- A merchant cannot load fake coins — they would draw their own credit, which cost them real sats to fund
- A merchant cannot load more than they have funded — the node enforces the ceiling
- Over a complete load-redeem cycle the merchant's credit position returns to its starting state — the only net change is that sats moved between a payer and a redeemer via the coin
- The Mint treasury always holds the backing — the merchant credit line is the mechanism, not the custody

### Anonymous Loads

The load action is deliberately anonymous. Anyone with a Lightning wallet can pay to load sats onto a coin at a merchant terminal. The system does not record who paid. This is consistent with the bearer instrument philosophy — the coin is cash. Cash doesn't record who filled it.

The merchant credit system provides accountability at the merchant level without requiring payer identity at the transaction level.

---

## Verification

A coin can be verified at two levels:

### Level 1 — NFC (Universal)

Any NFC-capable phone can tap the coin and open the holder verify page. No app required. The NDEF URL written to the chip at enrolment opens automatically in the phone's browser. The page shows:

- Current balance and cap
- Capacity percentage (fill bar)
- Status
- Load and redeem counts

This is the primary verification method for everyday use. Fast, zero friction, works with any modern smartphone.

### Level 2 — Optical PUF (Terminal)

The Sunmi terminal with the capture shoe can verify the coin's physical fingerprint against the enrolled ORB descriptors. This provides a secondary layer of authentication against physical counterfeiting — a perfect copy of the NFC UID alone is not sufficient if the physical surface texture doesn't match.

Optical verification is performed at the merchant terminal during load and redeem transactions if the capture shoe is present. It is not required for basic operation.

---

## The Payment Flow

### Loading a Coin

1. Merchant taps coin to Sunmi terminal — NFC poll returns UID
2. Terminal fetches coin record from node — shows current balance and available capacity
3. Merchant or coin holder enters load amount
4. Node validates: coin status, bucket capacity, merchant credit available, mint reserve headroom
5. Node calls Blink API — creates Lightning invoice for exact amount, memo encodes coin UID and merchant ID
6. Terminal displays QR code of invoice
7. Payer (anyone) scans QR with any Lightning wallet and pays
8. Blink sends webhook to node on settlement
9. Node updates: coin balance increases, merchant credit deployed increases, merchant credit available decreases, mint total loaded increases
10. Terminal shows confirmation

The payer is anonymous. The node records the settlement but not the payer identity.

### Redeeming a Coin

1. Holder presents coin at Sunmi terminal
2. Merchant taps coin — NFC poll returns UID
3. Terminal fetches coin record — shows current balance
4. Holder or merchant confirms redemption amount (partial or full)
5. Merchant enters Blink write key via the settings screen — key is held in terminal memory only
6. Node generates LNURL-withdraw — encodes coin UID and redemption amount
7. Terminal displays QR code of LNURL
8. Holder scans QR with their Lightning wallet
9. Holder's wallet calls LNURL endpoint, receives invoice, sends back to node
10. Node calls Blink API with write key — pays invoice from Lightning wallet
11. Write key is cleared from memory immediately after use
12. Node updates: coin balance decreases, merchant credit restored, mint total loaded decreases
13. Terminal shows confirmation, prints receipt

### Top-Up (Merchant Credit)

1. Merchant taps "Top Up Credit" on settings screen
2. Merchant enters amount in sats
3. Node generates Lightning invoice via Blink treasury wallet — memo encodes merchant ID
4. Terminal displays QR code
5. Merchant scans QR with their own Lightning wallet and pays
6. Blink webhook fires to node
7. Node updates: merchant credit line increases, merchant credit available increases
8. Terminal shows updated credit balance

---

## The Distributed Ledger

### Node Requirements

A compliant mint must run a minimum of three nodes, each holding an independent copy of the SQLite ledger. Nodes must be:

- Held by different keyholders — no single person should control more than one node
- Physically separated — different locations, not colocated
- Continuously replicated — Litestream WAL replication over SSH is the reference implementation

### Replication

All nodes replicate continuously. Each node's database is the source of truth for requests it serves. Litestream replicates the WAL log to peer nodes in near-real time. In the event of a primary node failure, the Cloudflare Worker failover router directs traffic to the next healthy node automatically.

### Minimum Hardware

- Raspberry Pi 4 Model B (2GB RAM) or Raspberry Pi 5
- Raspberry Pi OS Lite 64-bit
- SD card (32GB+) or SSD
- Network connection

Solar-powered nodes are supported. Reference build: Victron MPPT controller, 12V battery, 100W flexible panel, buck converter to 5V. Power draw: 3–5 watts continuous.

### Cloudflare Tunnel

Each node exposes its API via Cloudflare Tunnel. This means no port forwarding, no exposed IP address, no public server. The node is only reachable through Cloudflare's infrastructure. The Cloudflare Worker serves as health-checking reverse proxy and failover router across all nodes.

---

## Reconciliation

The node runs an automatic reconciliation check every 15 minutes. The check compares:

- The sum of all `balance_sats` across active coins against `total_loaded_sats` in the mint record
- The Lightning wallet balance against declared reserves
- Merchant `credit_deployed_sats` against active coin balances per merchant

Discrepancies are logged to the `reconciliations` table and surfaced on the dashboard. A passing reconciliation confirms the ledger is internally consistent.

Reconciliation does not replace the public Proof of Reserves — that is performed by the operator and community members by independently verifying the on-chain declared balances against block explorer data.

---

## Governance Conventions

These are not enforced by the protocol software. They are conventions that the community has adopted to maintain trust and consistency.

### Pizza Day (22 May)

The annual cap is reviewed and updated each year on Bitcoin Pizza Day. The mint operator publishes the new cap before 22 May. Any community member may propose a cap value by opening an issue in the protocol repository. The operator sets the final value.

The convention exists because cap values need to reflect current Bitcoin exchange rates to remain practically useful. A cap that was appropriate when Bitcoin was at one price may be either too restrictive or too large a year later.

### Proof of Reserves

The mint operator publishes the on-chain addresses of the single-sig and multisig treasury wallets publicly in the admin panel and on the dashboard. Any community member or coin holder can verify the declared balances independently by looking up those addresses on a block explorer.

It is expected that community members — not just the operator — perform this verification periodically. The public dashboard provides the declared balances and direct links to mempool.space for each address.

### Mint Registration

A mint that implements this protocol should register its mint code (e.g. `ITA001`) in the protocol repository so that other implementations can recognise it. Registration is voluntary and requires only opening an issue with the mint code, community name, and region.

---

## What the Protocol Deliberately Does Not Define

**Denomination values.** Each community sets values appropriate to their local economy. The protocol provides the mechanism (annual cap) but not the value.

**Mint mark design.** The visual identity of a coin belongs to its community. The protocol requires a mintmark to exist but does not specify what it looks like.

**Coin body colour and material.** The protocol specifies dimensions and NFC type. The material choice — what plastic, what colour, what recycled feedstock — belongs to the mint.

**Community governance.** How a community makes decisions about their mint is not the protocol's concern. The protocol provides tools — the three-wallet treasury structure, the distributed ledger — that make governance easier to conduct honestly. It does not define the governance itself.

**Fiat currency.** The protocol is Bitcoin-native. Fiat display (e.g. GBP equivalent on the dashboard) is a convenience feature for operators and holders. The ledger records sats only.

---

## Security Properties

**Bearer instrument.** Possession is ownership. There is no registered owner. The coin can be given away, lost, or stolen like cash. The system does not provide recovery for lost coins.

**No single point of custody.** The write key that can move funds from the Lightning wallet lives only on the merchant terminal, in memory, for ten minutes. No server, database, or configuration file holds it. A compromised node cannot drain the wallet.

**Reserve guard.** The node software will not accept a reserve declaration that would make the system insolvent. This is a hard constraint, not a warning.

**Cap ratchet.** The annual coin cap can only increase, never decrease. The node rejects any cap update where the proposed value is less than the current cap, or where the cap year precedes the current cap year. This protects existing coin holders — a coin enrolled at a given cap is always redeemable up to that cap, regardless of future mint-level cap changes.

**No payer identity.** Load transactions are anonymous by design. The system records that a coin was loaded and by how much. It does not record who paid.

**Physical unclonability.** The ORB optical fingerprint provides a second authentication factor that cannot be reproduced by copying the NFC UID alone. A counterfeit coin with a cloned UID will fail optical verification.

---

## Reference Implementation

**PlasticCoin Core** — Node.js / Express / SQLite
Repository: `plasticcoin-core/`
Current version: v3.0.0

The reference implementation is the normative implementation. Any behaviour not specified in this protocol document but implemented in PlasticCoin Core should be treated as intended behaviour until the protocol is updated.

---

## Contributing

The protocol is open. Implementations are welcome.

Useful contributions:

- Optical calibration data — real-world similarity threshold values from prototype testing
- Solar node build documentation — photographs, wiring diagrams, enclosure designs
- Alternative NFC tag compatibility reports — which tags work, which don't
- Translations of the holder verify page and dashboard
- Cap governance proposals for Pizza Day

To register a mint, open an issue with your mint code, community name, and region.

---

## Licence

The protocol specification is in the public domain. No permission required to implement it.

The reference implementation (PlasticCoin Core) is MIT licensed.

---

*No central authority. No permission required. The protocol governs it.*
