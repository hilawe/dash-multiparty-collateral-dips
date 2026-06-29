<pre>
  DIP: TBD
  Title: Multi-Party Masternode Collateral
  Author: Hilawe Semunegus (hilawe)
  Special-Thanks: pshenmic, Michael (kxcd)
  Status: Draft
  Type: Informational
  Created: 2026-06-20
  License: MIT License
</pre>

## Table of Contents
- Abstract
- Motivation
- Background
- The four trust points
- Requirements
- Constraints
- Design space
- Comparison
- Recommendation
- Companion proposals
- Open questions
- Prior work and references
- Acknowledgements
- Copyright

## Abstract

A Dash masternode requires its collateral, 1000 DASH for a regular masternode and 4000 DASH for an
Evolution node, to be a single transaction output controlled by one party. This makes any pooled or
shared masternode custodial, because whoever holds the key to the collateral output can take the pool.
This Informational Dash Improvement Proposal (DIP) defines the problem of trustless, multi-party
(shared) masternode collateral, states the trust model and the requirements a solution must meet, and
surveys the design space from wallet-layer constructions that need no consensus change to
protocol-level primitives that do. Its purpose is to give the community and Dash Core a shared framing
and a basis for choosing among the standards-track proposals that build on it. It records the authors'
recommendation while documenting the alternatives, so that the choice is an informed one.

## Motivation

DIP-0026 (multi-party payouts) lets a masternode's block reward be split among multiple payee
addresses on-chain, with no intermediary holding the funds in transit. That addresses trustless reward
distribution. It does not address trustless collateral pooling, which is the harder half of a
non-custodial alternative to services that pool collateral on a user's behalf (for example, services
that aggregate small deposits into masternode collateral).

Pooling matters for three reasons:

- it improves accessibility, because a user with less than a full collateral amount cannot run a
  masternode alone;
- it lowers reward variance, because a single shared node smooths the lumpiness of solo rewards;
- it reduces custody risk, because an intermediary that holds pooled funds carries the classification
  and counterparty risk that a non-custodial design avoids.

The idea of shared or "decentralized" masternode collateral has been discussed in the Dash community
since at least 2016 and has never shipped, because the core obstacles are real. This document names
them precisely so that a solution can be evaluated honestly rather than assumed.

## Background

A masternode is registered by a provider registration special transaction (ProRegTx) defined in
DIP-0003 (Deterministic Masternode Lists). The collateral is a single unspent transaction output
(UTXO) of exactly the required amount, referenced by the registration as an outpoint. The collateral
may be created by the registration transaction itself (internal collateral) or may be a pre-existing
output (external collateral). For external collateral, control is proven by an Elliptic Curve Digital
Signature Algorithm (ECDSA) signature from the collateral key over a defined message string, not by a
signature over the registration transaction.

Two facts follow that shape every design below. First, spending the collateral UTXO removes the
masternode from the deterministic list at that block, so the collateral output is both the bond and the
liveness anchor, and the two cannot be separated. Second, Evolution nodes (DIP-0028) use the same
single-outpoint collateral model at 4000 DASH.

DIP-0026 has been merged into Dash Core (the develop branch) and will activate with the V24 network
upgrade. The merged implementation caps the owner reward split at eight payees. That cap is relevant to
any design that would settle pooled rewards directly on-chain.

## The four trust points

A shared masternode concentrates four distinct powers. Each is a separate trust point. Solving one
does not solve the others, and discussion of the topic frequently conflates them.

1. Collateral custody. Whoever can spend the collateral UTXO can take the entire pool. On Dash,
   spending that UTXO also removes the masternode, so custody and node liveness are coupled.
2. Reward payout. Where the block reward goes. This is the piece DIP-0026 addresses. It is a
   settlement mechanism only.
3. Owner-key control. The owner key can change the payout list (including DIP-0026 shares) and the
   voting key through an update registration transaction. Even with trustless collateral, whoever
   holds the owner key can redirect future rewards.
4. Operator and proof-of-service. The operator runs the node. If it goes offline the node is banned by
   proof-of-service (PoSe) and stops earning. The operator cannot take the collateral but can degrade
   everyone's returns.

A trustless shared masternode is one that neutralizes points 1 and 3, no unilateral theft of principal
and no unilateral redirection of rewards, and makes point 4 accountable. Point 2 is a separate, already
addressed piece. Any design that secures only the collateral and forgets the owner key has not removed
the ability to steal future rewards.

## Requirements

Functional requirements, the properties a retail-scale pooling product needs:

- R1. Contributions smaller than the full collateral (accessibility).
- R2. Lower reward variance through pooling.
- R3. Decouple a funder's contribution from any single node.
- R4. Enter and exit without forcing a full node teardown.
- R5. Funder-chosen withdrawal timing, without creating dust outputs.
- R6. Governance participation for small funders.
- R7. Scale past the DIP-0026 payee cap for a churning client base.

Security requirements:

- S1. No single party can spend the pooled collateral to itself.
- S2. No single party can redirect the rewards.
- S3. Every funder can reclaim its principal without the cooperation of any party that could otherwise
  block it.
- S4. The construction is safe under chain reorganizations and mempool races.
- S5. One malicious or absent funder cannot destroy the node and everyone's collateral.

R1, R3, R4, and R7 are fundamentally about scale and churn, and they are what make the problem hard.

## Constraints

The following constraints are properties of Dash as deployed and bound the design space.

- The collateral is a single outpoint of a fixed amount, and spending it removes the node. Custody and
  liveness are the same output.
- On-chain pay-to-script-hash (P2SH) multisignature is limited in practice to roughly fifteen keys, by
  the maximum script-element size that bounds the redeem script. A large, churning client base cannot
  be represented as on-chain multisignature signers.
- Dash Layer 1 uses ECDSA only; it has no Schnorr or Taproot. A threshold signature therefore requires
  off-chain multi-party computation (MPC), or on-chain multisignature.
- Dash Script has no covenant or output-introspection operations today. It does provide the absolute
  and relative timelocks OP_CHECKLOCKTIMEVERIFY and OP_CHECKSEQUENCEVERIFY, and it has unused NOP
  opcodes into which a new operation could be introduced.

## Design space

Four families of approach, ordered by how invasive they are.

1. **Wallet layer, no consensus change.** A small n-of-m multisignature collateral with pre-signed
   refund transactions, the pattern used by existing shared-masternode services, optionally with
   signature-hash flexibility on the refunds. This is genuinely trustless for a small, known set of
   co-owners, and it works today. It does not scale, because of the roughly fifteen-key limit, and its
   refunds are static. It is useful as a product for co-owners, but it is not a retail-scale solution.
2. **Native multi-funder collateral (consensus change).** Extend the masternode registration rules so
   that the collateral may be an aggregate of several funders' outpoints, each independently
   refundable, with the reward weighted by contribution. This represents many funders natively. The
   hard part is allowing a funder to exit without removing the node, and doing so safely under
   reorganizations.
3. **Covenant opcode (consensus change).** Introduce a general-purpose operation, in the style of the
   Bitcoin proposal BIP-119 OP_CHECKTEMPLATEVERIFY (CTV), that lets an output commit to an exact
   spending template. The collateral can then be locked so that consensus, rather than a
   multisignature or a set of pre-signatures, guarantees each funder's refund. Because funders become
   named beneficiaries in a committed template rather than signers, this removes the fifteen-key limit.
   The operation can be shaped as a soft fork by occupying an unused NOP opcode, though it would still
   activate through a coordinated network upgrade.
4. **Layered design (Layer 1 and Layer 2).** Dash Platform holds the membership and per-client
   accounting for a churning base. Layer 1 holds the collateral, through one of the consensus
   mechanisms above, and settles rewards periodically via DIP-0026. This is necessary for retail
   scale, but it does not by itself solve custody. The Layer 1 piece still has to.

## Comparison

| Approach | Consensus change | Removes 15-key limit (R1, R7) | Trustless principal (S1, S3) | Dynamic membership (R4) | Maturity |
|----------|------------------|-------------------------------|-------------------------------|--------------------------|----------|
| Wallet-layer multisig + pre-signed refunds | No | No | Yes, small set only | No (static) | Deployable today |
| Native multi-funder collateral | Yes | Partly | Yes | Needs atomic replacement | Design |
| Covenant opcode | Yes (soft-fork-shaped) | Yes | Yes | Only with recursive covenants | Design |
| Layered Layer 1 and Layer 2 | Depends on Layer 1 piece | Through Layer 2 accounting | Inherits the Layer 1 piece | Yes, on Layer 2 | Concept |

## Recommendation

The authors recommend the covenant opcode as the foundational primitive, for three reasons:

- It is the most direct answer to the scaling limit, because it removes funders as signers.
- It rests on well-studied prior art, principally BIP-119 and the vault proposal BIP-345 (OP_VAULT),
  rather than on a bespoke and untested mechanism.
- It admits a fixed-term, fixed-membership first version that is fully trustless for principal without
  recursive covenants, which are the most contested and highest-risk part of the design.

The native multi-funder approach is retained as the documented alternative. Dash Platform is the
natural home for the membership and accounting layer once a sound custody primitive exists, and may
allow the dynamic product to be reached through periodic non-recursive rollover rather than through
recursive covenants. This recommendation is a starting point for the community's decision, not a
foreclosure of it; the purpose of this document is to make that decision well-informed.

## Companion proposals

This Informational DIP frames the problem. Two standards-track proposals build on it:

- a general-purpose covenant-opcode specification, justified on its own merits beyond this use;
- a trustless shared-collateral specification that consumes the opcode and defines the
  masternode-registration changes needed to use covenant collateral.

The native multi-funder approach is carried here as the alternative so that it can be weighed against
the covenant route.

## Open questions

- A fixed-term, non-recursive first version versus recursive covenants for dynamic membership, which
  is the central trade.
- How much of the membership and accounting layer Dash Platform can carry, and whether periodic
  non-recursive rollover can deliver the dynamic product without recursive covenants.
- The owner-key problem (trust point 3): a threshold owner key, or consensus-immutable payout shares.

## Prior work and references

- DIP-0003, Deterministic Masternode Lists: https://github.com/dashpay/dips/blob/master/dip-0003.md
- DIP-0026, Multi-Party Payouts: https://github.com/dashpay/dips/blob/master/dip-0026.md
- DIP-0028, Evolution Masternodes: https://github.com/dashpay/dips/blob/master/dip-0028.md
- BIP-119, OP_CHECKTEMPLATEVERIFY: https://github.com/bitcoin/bips/blob/master/bip-0119.mediawiki
- BIP-345, OP_VAULT: https://github.com/bitcoin/bips/blob/master/bip-0345.mediawiki
- "Announcing Decentralized Masternode Shares (forthcoming)," Dash forum, 2016 (historical context).

## Acknowledgements

The covenant direction was suggested by pshenmic. Thanks also to Michael (kxcd). This document builds on the masternode and payout
designs of DIP-0003 and DIP-0026, and draws on the Bitcoin covenant literature cited above.

## Copyright

Copyright (c) 2026 Hilawe Semunegus and the document authors. Licensed under the MIT License.
