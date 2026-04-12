
# PlasticCoin

A Protocol for Transforming Waste into Value


---

Abstract

PlasticCoin is a physical-digital totem minting system that converts locally collected plastic waste and solar energy into tokenised value backed by a publicly auditable Bitcoin reserve. Each totem coin is a unique, injection-moulded object embedded with a secure chip and linked to a verifiable ledger entry. The system combines material provenance, energy transparency, and cryptographic accounting to create a product designed for community-scale economies.

PlasticCoin does not aim to replace national currencies. It introduces a parallel system optimised for local circulation, transparency, and environmental alignment.


---

1. Introduction

In a small agricultural town, a workshop processes plastic waste collected by residents and visitors. Using solar-powered equipment, this material is transformed into durable physical coins. Each coin represents a claim on a shared reserve and carries a verifiable record of its origin.

This paper describes the design, production, and economic model of PlasticCoin, along with the underlying philosophy: that value can be created through transformation rather than extraction, and that money can be made legible at the scale of a community.


---

2. Monetary Context

Most modern monetary systems rely heavily on controlled scarcity. Precious metals derive value from limited supply; fiat currencies from issuance control; digital assets such as Bitcoin from algorithmic constraints.

Scarcity is effective, but it has structural consequences. Systems built on scarcity tend to prioritise extraction, accumulation, and short-term optimisation. Economists describe this as high time preference—prioritising immediate gain over long-term sustainability.

PlasticCoin explores an alternative: a system where input materials are abundant, but value is constrained and verifiable. The abundance lies in the waste stream. The discipline lies in issuance and reserves.


---

3. System Overview

PlasticCoin consists of three integrated components:

1. Physical Coin
A durable object made from recycled plastic and embedded with a secure chip.


2. Digital Ledger
A publicly accessible record linking each coin to identity, provenance, and value.


3. Reserve Treasury
A Bitcoin-backed pool that guarantees redeemability.



Each coin represents a fixed claim on the reserve at the time of activation.


---

4. Materials and Manufacturing

4.1 Inputs

Each coin is produced from three primary inputs:

Recycled plastic waste (e.g. bottle caps, containers)

Solar energy captured on-site

Organic filler (e.g. ground olive stones, locally sourced)


Typical material composition per coin:

~30–50g recycled plastic

~5–10g organic filler


4.2 Process

1. Plastic is collected, sorted, and shredded


2. Material is melted using solar-powered electricity


3. The mixture is injection-moulded into coin form


4. A secure chip module is inserted


5. The coin surface is imaged and recorded



Estimated energy per coin:

~0.05–0.1 kWh (depending on batch size and equipment efficiency)

4.3 Environmental Position

The system does not claim to eliminate environmental cost. Instead, it:

Utilises already-extracted materials

Substitutes solar for fossil energy where possible

Extends material lifecycle through durable transformation



---

5. Coin Design

5.1 Physical Properties

Diameter: 40mm

Material: recycled plastic composite

Unique colour and texture

Embedded denomination marker 

Colour-coded inset NFC tag 

Each coin is visually and materially unique due to raw material input variability.

5.2 Digital Interface

Each coin contains a passive NFC chip:

No battery required

Readable by standard smartphones

Opens a web-based record (no app required)


Displayed data includes:

Denomination

Origin (time, location, batch)

Verification status

Reserve backing



---

6. Security Model

6.1 Dual Authentication

Each coin is secured through two independent mechanisms:

1. Cryptographic Identity (Chip)

Stores a unique identifier

Linked to ledger entry


2. Physical Fingerprint (Surface)

High-resolution image of surface microstructure

Acts as a physical unclonable function (PUF)


6.2 Material-Based Security

Recycled plastic forms a heterogeneous structure:

Mixed polymers

Variable dyes

Irregular cooling patterns


This produces a surface that is:

Highly complex

Non-repeatable in practice

Economically infeasible to replicate exactly


6.3 Verification

Chip verifies identity

Surface verifies authenticity

Ledger confirms status


A successful counterfeit would require:

Cloning the chip

Reproducing the exact surface structure

Matching the ledger entry


This combined requirement creates strong resistance to forgery.


---

7. The Four Proofs

Each PlasticCoin embodies four verifiable properties:

7.1 Proof of Work

Human labour in collection, processing, and minting is embedded in the object.

7.2 Proof of Energy

Manufacturing energy is derived from solar input and can be audited at the system level.

7.3 Proof of Reserves

Each coin corresponds to a defined allocation within a publicly auditable Bitcoin treasury.

7.4 Proof of Origin

Material composition and recorded data link the coin to a specific place and time.


---

8. Monetary Model

8.1 Issuance

Coins are minted in batches. For each batch:

A fixed number of coins is produced

A corresponding amount of Bitcoin is allocated to the reserve

A reserve ratio is maintained (target: 1:1 backing)


No coin is activated without reserve allocation.

8.2 Denomination

Denominations are discrete and colour-coded.
Value is assigned at activation, not manufacture.

8.3 Activation

Coins are initially issued in a dormant state (“sleepers”).

At activation:

The batch is registered as active

Denominations become live

Reserve backing is locked


This allows:

Coordinated issuance

Transparent accounting

Ceremonial or event-based activation


8.4 Redemption

Holders may redeem coins via the operator:

Coin is verified (chip + ledger)

Equivalent Bitcoin value is transferred

Coin is optionally retired or recirculated


8.5 Governance

The system is:

Transparent: all reserves publicly visible

Local: operated by a known community entity

Accountable: subject to social and economic oversight


Trust is not eliminated, but made visible and verifiable.


---

9. Use Case: Workshop Economy

9.1 Tourist Participation

Visitors:

Collect plastic during their stay

Participate in coin production

Receive dormant coins


After activation:

Coins hold real value

Serve as both currency and artefact


9.2 Local Circulation

Residents and merchants:

Accept coins for goods and services

Spend or redeem as needed


This creates:

A local transaction loop

Reduced reliance on external payment systems


9.3 External Flow

Unredeemed coins held by visitors result in:

Net inflow of value to the community

Conversion of waste into retained economic value



---

10. Extended Production Model

The same infrastructure supports additional products:

10.1 Sunglasses

Produced in off-season

Sold during peak tourism

Align material origin with use context


10.2 Plant Pots

Sold locally year-round

Integrated with agricultural activity


These products:

Share material inputs

Extend economic activity beyond tourism

Reinforce local production identity



---

11. Economic Properties

PlasticCoin is designed as a complementary local currency with the following characteristics:

Non-inflationary (reserve-backed issuance)

Locally circulating

Physically embodied

Digitally verifiable


It is not intended to replace national currencies, but to:

Strengthen local trade

Increase economic transparency

Capture value from waste streams



---

12. Limitations and Open Questions

12.1 Scalability

The model is optimised for small communities. Larger-scale deployment requires coordination between mints.

12.2 Trust Boundaries

While transparent, the system still depends on operator integrity.

12.3 Environmental Accounting

Full lifecycle analysis is required to quantify net environmental benefit.

12.4 User Experience

Edge cases (chip failure, damage, verification friction) must be addressed in implementation.


---

13. Conclusion

PlasticCoin demonstrates that money can be:

Materially grounded

Energetically transparent

Economically accountable


It reframes waste not as an endpoint, but as an input to value creation. It does not attempt to redesign the global financial system. Instead, it operates at the scale where trust, production, and exchange are visible: the local community.

The protocol is open. Any community with access to plastic waste, solar energy, and basic manufacturing tools can implement it.

The system proposes a simple idea:

Value does not need to be extracted.

It can be made—locally, visibly, and deliberately.
