# PlasticCoin Protocol

**Version 0.6 — Working Draft**

---

## Preamble

This protocol describes an open, permissionless standard for community-based physical currency, backed by Bitcoin, minted from reclaimed plastic. It is designed to start small, grow slowly, and replicate honestly. No central authority governs it. The protocol governs it.

---

## 1. Principles

- **Decentralisation** — any community can mint. No permission required.
- **Transparency** — every layer of the system is publicly verifiable.
- **Locality** — each mint serves its own community first.
- **Longevity** — slow trust accumulates more durably than fast adoption.
- **Simplicity** — the protocol must be replicable with modest resources.
- **Resilience** — no single person, device or organisation can destroy or corrupt the network.

---

## 2. The Coin

The coin consists of two parts.

**The Body**

- Moulded from reclaimed plastic using injection moulding.
- 40mm diameter — standardised across all mints.
- Body colour determined by plastic feedstock — not standardised across mints.
- The unique random distribution of the recycled plastic feedstock creates a Physical Unclonable Function (PUF) — a one-of-a-kind surface fingerprint enrolled at manufacture and forming the second verification layer.
- Mint mark pressed into the surface — unique to each issuing community.
- No serial number or printed identifier on the coin surface. The NFC UID is the sole digital identifier. A visible serial number would aid counterfeiting.

**The NFC Tag**

- NTAG213 NFC tag embedded in the body at manufacture.
- Encased in standardised coloured plastic indicating denomination, in ascending value order:

| Colour | Order |
|--------|-------|
| 🔴 Red | Lowest value |
| 🔵 Blue | |
| 🟢 Green | |
| ⚪ White | |
| ⚫ Black | Highest value |

- Tag recessed 0.5mm into the rear face — protects from handling damage, sits flush.
- At minting the verification URL is written to the tag as NDEF data via the NFC bridge. When any holder taps the coin to any NFC-enabled phone the browser opens automatically to the verification page. No app, no account, no permission required.

Denomination values are set once per year on **22 May** by protocol consensus. Each community may set values appropriate to their local context within the standard colour framework.

All coins are manufactured locally within the issuing community. Closed loop manufacture is a protocol requirement.

---

## 3. The Value Layer

Each coin is backed 1:1 by Bitcoin held in the mint treasury. The backing is established at minting and does not move while the coin circulates.

**Treasury structure — three wallets:**

- **Multi-sig wallet** — held collectively by trusted community signers. Primary reserve.
- **Single-sig wallet** — held by the community treasurer. Operational reserve.
- **Lightning wallet** — held by each mint merchant. One Blink wallet per merchant. Receives inflows at minting and pays outflows at redemption.

Every coin minted has the Bitcoin payment transaction hash recorded in the protocol ledger. This hash is visible on the public verification page and links directly to the on-chain transaction via mempool. Any holder of any coin can verify the treasury backing of that specific coin independently and without permission.

Proof of Reserves is maintained on-chain at all times. Treasury solvency is publicly verifiable by anyone.

---

## 4. Dual-Layer Verification

Every coin supports two independent verification layers. Either layer can be checked independently. Both layers are required for high-value transactions.

**Layer 1 — Digital Identity (NFC)**

- Tap the coin rear face to any NFC-enabled smartphone.
- The NTAG213 UID is read and checked against the protocol ledger.
- Returns denomination, status, issue date, mint code, and Bitcoin transaction hash.
- Suitable for routine low-value verification.
- Requires no app, no account, no permission.

**Layer 2 — Physical Identity (PUF optical matching)**

- The unique surface pattern of each coin's recycled plastic body is enrolled at manufacture using ORB feature matching.
- At verification a live camera capture is compared against the enrolled descriptor stored in the ledger.
- Confirms the physical object is the genuine enrolled coin — not a copy or substitute body.
- Required for high-value transactions.
- Runs in the browser via OpenCV.js — no specialist server infrastructure required.
- A counterfeiter must defeat both layers simultaneously — cloning the NFC UID and exactly reproducing the three-dimensional surface microstructure of the genuine coin.

**Verification tiers:**

| Tier | Method | Use case |
|------|--------|----------|
| NFC only | Any NFC smartphone | Routine low-value checks, holder self-verification |
| Full dual-layer | NFC + optical via Sunmi V2S | High-value transactions, merchant verification |

---

## 5. The Protocol Ledger

The protocol operates a shared, distributed, append-only ledger. This ledger is the record of truth for all coins minted across all participating communities. It is not hosted on any centralised platform. It lives across the node network.

**Each ledger entry records:**

- Mint code — unique identifier of the issuing community.
- Coin ID — the NTAG213 NFC UID (hex string, colon-separated).
- Denomination — the colour standard value.
- Denomination in satoshis — the precise Bitcoin value at time of minting.
- Lightning payment hash — from Blink at moment of minting, interim proof of payment.
- Settlement status — unsettled (in Lightning wallet) or settled (swept to on-chain treasury).
- Bulk transaction hash — on-chain treasury transaction hash, populated at sweep.
- Optical descriptors — the enrolled PUF fingerprint (keypoints and descriptors).
- Issue date and timestamp.
- Status — inactive or active.
- Cycle count — number of times this coin has carried Bitcoin value.
- Verify count and last verified timestamp.

**The ledger is:**

- Append only — no entry is ever deleted or modified.
- Replicated in real time across all nodes in the network.
- Publicly readable — anyone can query and audit.
- Write protected — new entries require threshold confirmation from node keyholders.
- Independent — no reliance on any third party platform or account.

---

## 6. The Node Network

The ledger is maintained across a distributed network of community-owned nodes. Each mint operates a minimum of three nodes.

**Node specifications:**

- Raspberry Pi 4 2GB or equivalent low cost hardware.
- Always on, headless operation.
- SQLite database — the local ledger instance.
- Litestream — real time replication across all nodes in the network.
- Read-only API endpoints — publicly accessible for dashboard and verification queries.

**Node ownership:**

- Each node is owned and physically held by a community keyholder.
- Node owners are the same persons as the multi-sig treasury wallet holders.
- Nodes are located in separate physical locations within the community.

**Optional: Solar powered node**

Nodes may be powered by solar. The reference hardware configuration is:

- Raspberry Pi 4 2GB
- Victron 75/15 MPPT controller
- 12v 35ah battery
- 12v to 5v buck converter with 5a fuse
- 10a fuse on battery output
- 100w semi-flexible PV panel with MC4 bulkhead connectors
- Weatherproof enclosure with MPPT controller mounted externally
- Bulkhead LAN port and WiFi for dual connectivity options

Solar nodes are identified in the ledger and displayed distinctly on the community dashboard node map. Solar operation eliminates mains dependency and enables placement anywhere with solar exposure.

**Network resilience:**

- Every mint adds a minimum of three nodes to the network.
- Data exists simultaneously across all nodes in all participating communities.
- To destroy the ledger an attacker must simultaneously gain physical access to every node across every community in the network — this becomes progressively harder with every new mint that joins.
- No single login, credential or individual can delete or corrupt the shared ledger.

---

## 7. Node Map

Each node exposes its public IP address to an IP geolocation service on startup and periodically thereafter. Resolved coordinates are stored in the ledger and surfaced on the community dashboard as a live node map.

Geolocation is accurate to city or region level — precise addresses are never exposed. The node map shows geographic distribution of the network, not exact locations.

Solar powered nodes are visually distinguished on the map. As the protocol grows across communities the map becomes a live representation of the network's global reach.

---

## 8. Reconciliation

The network performs automatic reconciliation on a regular cycle. Each reconciliation check verifies:

- Total coins minted per mint code match the ledger entries.
- All nodes hold identical database state.
- Treasury Proof of Reserves matches total value of active settled coins.
- Each merchant Lightning wallet net position matches expected balance — inflows from minting minus outflows from redemption.
- Sweep threshold status per merchant.

If all nodes agree — no action. Silent confirmation.

If any discrepancy is detected — all node owners receive an immediate system wide warning notification via the linked Android notification webapp.

Reconciliation is automatic, continuous and requires no human intervention during normal operation. It is a witness and verification layer, not a transaction approval layer.

---

## 9. The Lightning Flow

The merchant Blink Lightning wallet is the operational layer of the mint economy. It handles both sides of the coin lifecycle.

**Inflow — minting:**

- Buyer pays a Lightning invoice generated on the Sunmi V2S terminal.
- Sats arrive in the merchant Blink Lightning wallet.
- Coin is activated in the ledger with Lightning payment hash as interim proof.
- Settlement status: unsettled.

**Outflow — redemption:**

- Coin is returned physically to the merchant terminal.
- Merchant confirms redemption on the Sunmi V2S.
- A LNURL QR code is generated and displayed on the terminal screen.
- Customer scans the QR with any Lightning wallet.
- Sats flow out of the merchant Blink Lightning wallet to the customer.
- Redemption event recorded in the ledger — coin status returns to inactive.
- Redemption outflow visible on dashboard and reconciliation.

**Net position:**

- Net position = total sats received via minting − total sats paid out via redemption.
- Net position is the merchant's Lightning wallet balance attributable to coin activity.
- Sweep threshold is applied to the net position, not gross inflows.

---

## 10. The Sweep Process

The sweep transfers the merchant Lightning wallet net position to the on-chain treasury wallet, anchoring circulating coin value to the Bitcoin blockchain.

**Sweep triggers — whichever is reached first:**

- **Value threshold** — net position exceeds a defined multiple of the highest denomination coin value.
- **Time threshold** — maximum days between sweeps regardless of value.
- **Coin count threshold** — number of unsettled active coins exceeds defined limit.

Thresholds are set per mint and apply to all merchants within that mint. They are defined at mint setup and visible on the community dashboard.

**When a sweep occurs:**

- Merchant transfers net position from Blink Lightning wallet to on-chain treasury wallet.
- On-chain transaction hash is recorded in the ledger.
- All active coins minted since the last sweep are updated — settlement status moves to settled, bulk transaction hash recorded.
- A sweep event is written to the events log.
- Reconciliation runs immediately to confirm on-chain treasury balance matches total settled coin value.
- Dashboard sweep alert clears.

**Dashboard sweep monitoring per merchant:**

- Total active coins.
- Total unsettled coins and sats value — currently in Lightning wallet.
- Total settled coins and sats value — confirmed on-chain.
- Days since last sweep.
- Current Lightning wallet balance via Blink read-only API.
- Sweep required indicator — flagged when any threshold is breached.

**Verification page settlement display:**

For a settled coin: the verification page shows a link to the on-chain bulk settlement transaction on mempool.

For an unsettled coin: the verification page shows the Lightning payment hash as interim proof and notes that settlement to the community treasury is pending.

---

## 11. Minting

To mint a coin a community must:

- Establish a treasury with all three wallet layers funded.
- Publish Proof of Reserves before first coin is issued.
- Manufacture the coin locally from reclaimed plastic — closed loop production required.
- Enrol the coin's PUF optical fingerprint at manufacture using the standard ORB pipeline — minimum 200 keypoints required.
- Register the coin's NFC UID in the protocol ledger with full entry details.
- Generate a Lightning invoice on the Sunmi V2S terminal for the denomination value.
- Receive Lightning payment from buyer — sats arrive in merchant Blink wallet.
- Record Lightning payment hash in the ledger against the coin UID.
- Write the verification URL to the NFC tag as NDEF data via the NFC bridge.
- Mark the coin physically with the community mint mark.
- Confirm the coin's denomination colour matches the NFC tag standard.

The coin enters circulation only after all ledger entry steps are complete, replicated across the node network, and the NDEF verification URL has been written to the tag.

---

## 12. Merchant Hardware

The reference hardware for minting and full dual-layer verification is the **Sunmi V2S POS terminal**.

Each merchant within a mint operates one Sunmi V2S. The terminal handles:

- Lightning invoice generation — buyer pays at point of minting.
- Coin minting — NFC bridge reads UID, ledger entry created, NDEF URL written to tag.
- Full dual-layer verification — NFC tap plus optical PUF matching via capture shoe and rear camera.
- Lightning redemption — LNURL QR generated and displayed, customer scans with Lightning wallet.

**Onboarding a new merchant:**

- Install the SatsCASH webapp on the Sunmi V2S.
- Attach the standard capture shoe to the terminal.
- Configure the merchant's Blink Lightning wallet.
- Register the merchant in the protocol ledger under the mint code.
- Set Node URL in the webapp settings to point to the PlasticCoin Core node.
- Confirm full ledger sync and test mint with a prototype coin.

Each new merchant terminal extends the mint's reach into the community without requiring any changes to the protocol or the node network. Merchants are participants in a mint, not independent mints.

---

## 13. Verification

**NFC verification — any smartphone:**

- Holder taps coin to any NFC-enabled phone — browser opens automatically to the verification page.
- The UID is read and checked against the protocol ledger.
- Denomination, status, issue date, mint identity, and settlement status are returned.
- For settled coins — mempool link to on-chain bulk transaction hash.
- For unsettled coins — Lightning payment hash as interim proof.

**Full dual-layer verification — Sunmi V2S with capture shoe:**

- NFC verification completed first via the terminal.
- Live camera capture of coin front face taken under controlled flash illumination through the capture shoe.
- OpenCV.js extracts ORB descriptors from the live image in the browser.
- Descriptors compared against the enrolled PUF fingerprint in the ledger.
- Similarity score returned — pass threshold determined empirically during prototype calibration and documented per mint.
- VERIFIED only if both NFC and optical layers pass.

Verification requires no app, no account and no permission at any tier.

---

## 14. Circulation

Once minted the coin circulates physically. No ledger entry is created when a coin changes hands. The physical transfer is the transfer. The value is in the object.

Coins operate in a closed loop — they leave the mint, circulate within the community, and return to the issuing mint. Each coin is the sovereign responsibility of its issuing mint for its entire circulation life. Redemption at a mint other than the issuing mint is not supported.

---

## 15. Redemption

A coin is redeemed by returning it physically to a merchant terminal within its issuing mint. The merchant:

- Taps the coin to the Sunmi V2S NFC reader.
- Verifies the coin is active in the protocol ledger.
- Confirms redemption on the terminal.
- A LNURL QR code is generated and displayed on the Sunmi V2S screen.
- Customer scans the QR with any Lightning wallet.
- Sats flow out of the merchant Blink Lightning wallet to the customer instantly.
- Ledger records the redemption — coin status returns to inactive, outflow amount and timestamp recorded.
- Merchant net position decreases by the redeemed denomination value.
- Dashboard and reconciliation updated.

Any merchant within the issuing mint may redeem any coin issued by that mint. The issuing mint is collectively responsible for honouring all redemptions.

---

## 16. The Community Dashboard

Each mint operates a public read-only community dashboard hosted on Cloudflare Pages. The dashboard surfaces live data from the node network and treasury wallets, making the protocol's transparency principle human readable and publicly accessible.

**The dashboard displays:**

- Total coins in circulation — broken down by denomination and merchant.
- Treasury Proof of Reserves — live Bitcoin balance for multi-sig and single-sig wallets.
- Each merchant's Blink Lightning wallet balance — live, via Blink read-only API.
- Per merchant sweep status — net position, unsettled coins, days since last sweep, sweep required flag.
- Lightning flow summary — total inflows from minting, total outflows from redemption, net position per merchant.
- Recent events — coins minted, redeemed, swept, with timestamps.
- Node network status — online or offline per node, last sync timestamp, solar status.
- Node map — live geolocation of all nodes across the network.
- Reconciliation status — last check result, any active warnings.

**How it works:**

- Static webapp hosted on Cloudflare Pages — fast, globally distributed, low cost.
- Queries the node network via read-only public API endpoints — no credentials required.
- Queries treasury wallets via public on-chain data.
- Queries each merchant Blink wallet via Blink read-only API.
- Never writes anything — purely read-only at every data source.
- If dashboard hosting goes down the underlying node data is completely unaffected.

The dashboard is the protocol's public face. It makes trust observable.

---

## 17. Mint Closure

If a mint ceases operation it must:

- Publish notice with a minimum 90 day redemption period.
- Honour all redemptions during that period.
- After closure the multi-sig wallet holders retain custody of any unredeemed treasury Bitcoin indefinitely.
- The ledger entries for that mint code remain permanently readable across the node network.
- The mint code and mint mark retire permanently and may not be reused.

---

## 18. Disputes

No central arbiter exists. Three principles apply:

- **Verify first** — tap the coin, the browser opens, the ledger responds. Most disputes resolve immediately.
- **The protocol ledger is the record of truth** — replicated, append only, tamper evident, owned by no single party.
- **The treasury is public** — solvency disputes resolve on-chain via Proof of Reserves and the community dashboard.

Mint reputation is the primary enforcement mechanism. The protocol provides the tools for verification. The community provides the trust.

---

## 19. Replication — Starting a New Mint

Any community may adopt this protocol and establish a new mint. To do so they must:

- Establish a unique mint code — registered in the protocol ledger.
- Create a unique mint mark — pressed into every coin they issue.
- Set up the three wallet treasury structure and publish Proof of Reserves.
- Deploy a minimum of three nodes — one per community keyholder.
- Connect nodes to the protocol network and confirm full ledger sync.
- Procure and configure at least one Sunmi V2S terminal with capture shoe.
- Configure each merchant's Blink Lightning wallet and register in the ledger.
- Set sweep thresholds appropriate to local denomination values.
- Deploy the community dashboard.
- Set local denomination values within the standard colour framework on 22 May.
- Source reclaimed plastic feedstock and establish local coin manufacture.
- Start small — low face values, limited initial supply, build trust before scaling.

No existing mint authorises a new mint. The protocol authorises it.

---

## 20. What the Protocol Does Not Define

- The specific face values of denominations — set locally within the colour standard on 22 May each year.
- The design of the mint mark — belongs to each community.
- The body colour of the coin — determined by plastic feedstock.
- The governance structure of community treasury signers — determined locally.
- The size or composition of the local community — determined locally.

The protocol defines the structure. The community defines the expression.

---

## Status

**Draft v0.6** — April 2026

**Pending items for future versions:**

- Mint code registration process — how a new mint registers its code on the shared network.
- Node onboarding procedure — step-by-step for joining an existing network.
- Denomination setting governance — the process by which communities agree values on 22 May.
- Optical similarity threshold calibration standard — empirical values from prototype testing to be documented here.

---

## Reference Implementation

The reference implementation of this protocol is **PlasticCoin Core** — a Node.js server designed to run on Raspberry Pi 4 hardware.

- API server: Express.js
- Database: SQLite with WAL mode (Litestream replication)
- Lightning: Blink wallet API
- Dashboard: Static HTML hosted on Cloudflare Pages

Repository: [github.com/plasticcoin/core](https://github.com/plasticcoin/core)

---

*PlasticCoin Protocol is open. Anyone may implement it. No permission required.*
