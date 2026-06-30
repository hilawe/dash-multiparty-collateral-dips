<pre>
  DIP: TBD (Standard; number assigned by DIP editors on submission)
  Title: Output-Template Covenant Opcode (OP_CHECKTEMPLATEVERIFY)
  Author: Hilawe Semunegus (hilawe), pshenmic
  Special-Thanks: Michael (kxcd)
  Status: Draft
  Type: Standard
  Layer: Consensus (soft-fork-shaped; ships via a coordinated network upgrade)
  Created: 2026-06-20
  License: MIT License
</pre>

Status of this draft. This is a concrete proposal, not yet a final specification. The opcode (named
OP_CHECKTEMPLATEVERIFY, a working name, after its Bitcoin antecedent) and the template-hash fields are
proposed below, and the DefaultTemplateHash preimage is now specified byte-exact. The remaining open
items are a reference implementation with published test vectors (the spec reduces but does not by itself
eliminate the risk of divergent hashes), the recursion stance (analyzed and left open), and the full
reorg and replay analysis. These should be closed before any formal submission.

## Table of Contents
- Abstract
- Motivation
- Specification
- Rationale
- Backwards compatibility
- Deployment
- Reference implementation
- Open design questions
- Acknowledgements
- Copyright

## Abstract

This DIP proposes a covenant opcode for Dash Script that lets an output commit to an exact template
of the transaction that may spend it. A spend is valid only if the spending transaction matches the
committed template (its outputs, and enough of its structure to prevent malleability). The opcode is
general-purpose and enables vaults, congestion-controlled payments, payment pools, and non-custodial
shared collateral, among other constructions. It is designed to occupy a currently unused NOP opcode
so that nodes which have not upgraded treat it as a no-op.

## Motivation

Dash Script today cannot constrain where a coin may be sent, so any party with the spending key may send
a UTXO anywhere. Many useful constructions require the opposite, an output that can only be spent in a
pre-committed way:
- Vaults: funds that can only move to a pre-agreed recovery or cold-storage path.
- Payment pools and congestion control: commit to a fan-out of payments and realize it later.
- Non-custodial shared collateral (the companion application DIP): a pooled masternode collateral that
  can only ever be spent to refund its funders.

These are achievable today only by pre-signing transactions and then deleting keys, which is fragile
(key management, static membership, no consensus guarantee). A covenant opcode makes the guarantee a
consensus rule. This DIP is justified on this general utility; the shared-collateral application is one
consumer, not the sole rationale.

This DIP specifies a script-layer primitive only. The applications that consume it, and any product or
interface layer above them, are out of scope and are covered by their own proposals.

## Specification

OP_CHECKTEMPLATEVERIFY redefines OP_NOP4 (0xb3). When executed:

- it reads a 32-byte value (the StandardTemplateHash) from the top of the stack;
- it computes the DefaultTemplateHash of the spending transaction at the current input index;
- if the two are unequal, script execution fails; otherwise execution continues with the stack
  unchanged (the opcode does not pop its argument, which preserves NOP-upgrade semantics).

DefaultTemplateHash adapts the BIP-119 construction to Dash's transaction serialization. It is a single
SHA-256 over the concatenation of the following fields, in this exact order, every integer
little-endian:

1. the 32-bit packed version field, 4 bytes, exactly as Dash serializes it, which is
   `nVersion | (nType << 16)` (`src/primitives/transaction.h`). This commits both the version and the
   transaction TYPE in one field.
2. nLockTime, 4 bytes (uint32).
3. inputCount, 4 bytes (uint32), the number of inputs.
4. sequencesHash, 32 bytes, a single SHA-256 of the concatenation of the 4-byte (uint32) nSequence of
   every input, in input order.
5. outputCount, 4 bytes (uint32), the number of outputs.
6. outputsHash, 32 bytes, a single SHA-256 of the concatenation of every output serialized as Dash
   serializes CTxOut (8-byte little-endian amount, then a CompactSize scriptPubKey length, then the
   scriptPubKey bytes), in output order.
7. inputIndex, 4 bytes (uint32), the index of the input being spent.

Two fields are deliberately NOT committed, and the reasons are Dash-specific.

- The scriptSig of any input. Dash has no SegWit, so a script-path spend uses P2SH and carries its
  redeemScript and satisfier in the scriptSig. Committing the spending input's scriptSig would be
  circular, because a Path-A signature is not known when the template hash is fixed at funding. For a
  single-input spend, and the collateral refund is exactly that, a committed inputCount of one already
  prevents any input from being added, so omitting the scriptSig does not let value be redirected. The
  cost is that the spend transaction ID is malleable (the scriptSig is part of the txid on a no-SegWit
  chain), so the OUTPUTS are fixed but the parent txid is not. Any design that needs a stable parent
  txid, for example a prebuilt CPFP child, must address this separately (see the fee strategy in the
  companion application DIP). Multi-input templates are out of scope for the first version.
- vExtraPayload. Dash special transactions append a payload, serialized only when the type is not
  TRANSACTION_NORMAL (`src/primitives/transaction.h`). Field 1 commits the packed version and type, so a
  template that fixes a normal-type spend cannot be turned into a special-type transaction carrying a
  payload without changing the committed field and failing the check. Committing the version/type field
  is therefore sufficient, and the payload does not need a separate commitment for the normal-type refund.

Hashing primitive. Each sub-hash (fields 4 and 6) and the outer hash are a single SHA-256, matching
BIP-119 rather than Dash's double-SHA-256 txid convention, so the construction stays close to the
reviewed BIP-119 design. This choice, and every field width above, must be locked by a reference
implementation and a published set of test vectors before submission. The specification reduces the risk
of two implementations computing different hashes, but only test vectors eliminate it.

Size and denial-of-service bound. outputsHash and sequencesHash are each linear in the transaction and
are already bounded by the 100,000-byte standard transaction size (`src/policy/policy.h`), so a single
OP_CHECKTEMPLATEVERIFY adds no super-linear validation cost. The application DIP additionally bounds the
refund fan-out (maximum funder count and minimum contribution) so a large pool cannot approach the
standard-size limit.

Argument length. A 32-byte argument triggers the verify. An argument of any other length is treated as
a no-op, which reserves the opcode for future extensions, following BIP-119.

Standardness. Once activated, spends that satisfy the opcode are standard for relay and mining, with a
mempool policy that mirrors BIP-119.

Open at this layer (see Open design questions): the recursion stance (analyzed under Recursion, left
open) and the full reorg and replay analysis, including each step of any bounded nesting.

## Rationale

An exact-template commitment (CTV-style) is preferred over general output introspection (Elements-style
OP_INSPECT* opcodes). An exact commitment is simpler, has a smaller attack surface, is bounded by
construction (explained under Recursion), and has years of review behind it (BIP-119). General
introspection is more expressive but a much larger and more contentious consensus surface.

Alternatives considered, all documented in the companion Informational DIP:

- pre-signed transactions with key deletion (no consensus change, but fragile and static);
- threshold signatures via off-chain multi-party computation (no new opcode, but a federation and heavy machinery);
- general introspection opcodes (more power, more risk).

## Recursion

An earlier draft called this opcode "non-recursive" as a safety property. That was imprecise, and this
section states the position the maintainers should weigh, with the choice left open rather than
pre-decided. Three distinct things get conflated in the covenant-recursion debate, and they have very
different risk.

1. Unbounded recursion. A covenant that can force its successor to also be a covenant, computed from
   runtime data, so the encumbrance propagates indefinitely. This is what the Bitcoin community objects
   to most, and it is what general introspection opcodes (OP_CAT-based constructions, Elements-style
   introspection) can express. This opcode does NOT enable it, and that is a property of an exact-hash
   commitment, not a rule that has to be added. A spend's template commits to the concrete scriptPubKey
   bytes of every output. To pay into another covenant, the parent template must already contain the
   child's exact P2SH script, which contains the child's exact template hash, and so on. The structure
   must be computed bottom-up and must terminate, so no unbounded or self-referential family can exist.
2. Bounded, pre-committed nesting. A committed template pays into another concrete covenant output whose
   template hash is fixed when the parent is created. The opcode DOES permit this, because the parent
   simply commits to a child P2SH script like any other output script. It enables a pre-planned rollover
   schedule (a fixed sequence of term renewals or membership states agreed at funding), but nothing
   decided after funding, since every step's hash must be known in advance. The encumbrance is finite and
   its full extent is visible at creation.
3. Cooperative replacement. A funder quorum signs a Path A spend whose output is a freshly agreed
   covenant. This needs no recursion rule at all, because it is an ordinary authorized spend to whatever
   the quorum agrees at the time. It delivers genuinely dynamic membership (join, leave, swap decided
   later) at the cost of requiring quorum liveness at each change.

The practical consequence for the shared-collateral application: dynamic membership does not require
consensus recursion. Cooperative replacement (3) achieves it with no nesting rule, trading autonomy for a
live quorum. So the real decision is narrow, whether to ALSO allow bounded nesting (2) for autonomous or
pre-planned rollover, on top of cooperative replacement.

Two defensible options, left open for the maintainers:

- Option A, forbid nested covenant outputs. Add a rule that a committed template's outputs may not
  themselves be recognized covenant scripts (enforced as a consensus check, or at minimum a standardness
  rule, on the template's output scripts). This makes the property "no nesting" true by rule rather than
  by hope, gives the simplest reasoning surface, and still supports membership change through cooperative
  replacement (3). The cost is no autonomous rollover, so node continuity past a term depends on quorum
  liveness.
- Option B, allow bounded nesting. Permit (2) and analyze it directly rather than claiming it away. The
  analysis can lean on the boundedness in (1): there is no infinite lock, and the full encumbrance path
  is fixed and inspectable at creation. The cost is a larger reasoning surface and the reorg, replay, and
  size questions below applied across each nested step, in exchange for pre-planned rollover without a
  live quorum at each step.

Whichever is chosen, two things hold. Unbounded recursion (1) is out of scope and not enabled by the
opcode. And each individual step must respect the same bounds as a single template, the 520-byte
redeemScript limit (`src/script/script.h`) and the 100,000-byte standard transaction size
(`src/policy/policy.h`). Those bound each step, not the total depth of a precomputed nested chain, which
can span many transactions. Total depth is instead bounded by the requirement that the whole structure be
computed and committed at creation, so it is finite and fully known up front, just not capped by the
per-transaction byte limits.

## Backwards compatibility

The change is soft-fork-shaped. OP_NOP4 is currently a no-op, so transactions using the new opcode are
non-standard but not invalid to un-upgraded nodes, which limits chain-split risk. This does NOT mean no
upgrade is required. Dash ships consensus changes through coordinated network upgrades (the EHF-style
V-series activations), and this opcode would activate the same way. The benefit of the NOP-upgrade
shape is a smaller blast radius, not avoidance of the coordinated activation.

## Deployment

Via a coordinated network upgrade (a new deployment in the V-series), gated behind masternode
signalling, consistent with how recent Dash consensus features (extended addresses, and similar) have
shipped.

DIP-0026 multi-party payouts is now merged into Dash Core and activates with the V24 network upgrade,
so the proposed target for this opcode is the coordinated upgrade after V24. Payouts deliver the
non-custodial reward-distribution half, and this opcode is the primitive
that makes the non-custodial collateral half practical, so sequencing it as the next upgrade after
payouts gives builders the full foundation for shared-masternode and other pooled products. Because the
opcode is general-purpose, it stands on its own merits and is not gated on the shared-collateral
application.

## Reference implementation
To be written after the Specification is pinned. The expected surface is the script interpreter
(src/script/interpreter.cpp), the opcode tables (src/script/script.h), a new transaction-template
hasher, standardness and policy rules, and tests (unit and functional).

## Open design questions
- A reference implementation and published test vectors for the DefaultTemplateHash. The preimage is now
  specified byte-exact under Specification (field order, integer widths, little-endian encoding, the
  packed version/type field, CompactSize scriptPubKey lengths, output ordering, the single-SHA-256
  sub-hashes, and the deliberate scriptSig and vExtraPayload omissions). Test vectors are what finally
  guarantee two implementations agree.
- The non-SegWit consequence that omitting the scriptSig from the commitment leaves the spend
  transaction-ID malleable, which breaks prebuilt CPFP children and watchtower tracking. Pin the exact
  Path-A and Path-B scriptSig forms, or do not rely on CPFP children of the spend. (Addressed in the
  application DIP's fee strategy.)
- Argument-length / future-extensibility policy.
- The recursion rule (analyzed under Recursion, left open for the maintainers). Unbounded recursion is
  out of scope and not enabled by an exact-hash commitment. The open choice is Option A (a rule that
  forbids a covenant output inside a committed template) versus Option B (allow bounded, pre-committed
  nesting and analyze it directly). Dynamic membership does not depend on this choice, because cooperative
  quorum replacement delivers it without nesting.
- Full reorg, replay, and denial-of-service analysis, including interaction with Dash special
  transactions and the masternode collateral-removal rule.
- Standardness and mempool policy.

## Acknowledgements
Thanks to Michael (kxcd). This draws on prior art, principally BIP-119 OP_CHECKTEMPLATEVERIFY (Jeremy
Rubin) and OP_VAULT (BIP-345).

## Copyright
This document is licensed under the MIT License.
