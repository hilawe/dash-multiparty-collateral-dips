<pre>
  DIP: TBD (Standard; number assigned by DIP editors on submission)
  Title: Trustless Shared Masternode Collateral
  Author: Hilawe Semunegus (hilawe)
  Special-Thanks: pshenmic, Michael (kxcd)
  Status: Draft
  Type: Standard
  Layer: Consensus / Applications
  Requires: the Output-Template Covenant Opcode DIP; DIP-0003; DIP-0026
  Created: 2026-06-20
  License: MIT License
</pre>

Status of this draft. This is a design sketch, not a finished specification. The fixed-term,
fixed-membership MVP still has open items, the covenant template hash, the byte-exact committed
parameters, the renewal path, the fee strategy, and the reorg/replay rules. An independent review
(2026-06-27) found a reward-redirection attack on the funding step. The authority model below addresses
it, and the funder authorization is enforced by consensus at registration, not by an off-chain document.
Funding and registration are two separate transactions and funders sign only the first, so a manifest
that funders merely verify before funding would not bind the later registration. Do not treat the
properties in the Trust analysis as established until the open items close.

## Table of Contents
- Abstract
- Motivation
- Specification: the covenant, funding, registration, operation, refund
- Trust analysis
- Rationale
- Backwards compatibility
- Limitations and future work
- Open design questions
- Acknowledgements
- Copyright

## Abstract

This DIP specifies a non-custodial shared masternode collateral. Multiple funders pool the required
collateral (1000 DASH for a masternode, 4000 for an Evolution node) into a single output locked by a
covenant (the Output-Template Covenant Opcode DIP) that guarantees, by consensus, that the collateral
can only ever be spent to refund the funders their contributions. Rewards are split among funders via
DIP-0026. The first version is fixed-term and fixed-membership, and is fully trustless for principal
without recursive covenants. Dynamic membership and indefinite life are deferred (see Limitations).

## Motivation

DIP-0026 made reward distribution non-custodial but left collateral custody custodial. This DIP closes
that gap for a known set of funders, removing the need for a trusted operator to hold the pooled
collateral. It is the consensus-enforced successor to the wallet-layer pattern (multisig plus
pre-signed refunds) used by existing shared-masternode services, and it is the building block for a
retail-scale, Platform-accounted product. See the companion Informational DIP for the full design
space and trust model.

This DIP specifies the consensus mechanics of the pooled collateral and its settlement. The funder-facing
application, including the signed-action interface (R9 in the Informational DIP) and the compounding
behavior (R8), is a product and Platform layer above this primitive and is out of scope here.

## Specification

### The covenant
The pooled collateral is a single output paying P2SH of the following redeemScript. `refundHashA` and
`refundHashB` are the covenant commitments (per the opcode DIP) to the exact refund outputs, one output
per funder paying their contribution back to their address, one hash per spend path (see below). The
script also commits the masternode registration
parameters, so that registration cannot deviate from what the funders agreed (see Registration). The
committed parameters are pushed and immediately dropped, so they do not affect spend execution but are
recoverable by parsing the redeemScript. Two spend paths, both forced to the same refund template, so
neither can redirect funds:

```
# Committed registration parameters (parsed at registration; dropped at spend).
<ownerKeyID> OP_DROP
<votingKeyID> OP_DROP
<operatorKey> OP_DROP
<payoutSharesHash> OP_DROP
<operatorReward> OP_DROP
OP_IF
    # Path A: cooperative early dissolve, authorized by an M-of-N committee of
    # funder representatives, but funds are still forced to the refund outputs.
    <M> <controlPubKey_1> ... <controlPubKey_N> <N> OP_CHECKMULTISIGVERIFY
    <refundHashA> OP_CHECKTEMPLATEVERIFY
OP_ELSE
    # Path B: permissionless backstop after the term ends.
    <termEndLockTime> OP_CHECKLOCKTIMEVERIFY OP_DROP
    <refundHashB> OP_CHECKTEMPLATEVERIFY
OP_ENDIF
```

The two branches commit to TWO refund-template hashes, not one. They bind the SAME refund outputs (one
output per funder) but different transaction-level fields, because OP_CHECKTEMPLATEVERIFY commits the
spending transaction's nLockTime and input sequences. Path A is the early dissolve and its template
(`refundHashA`) commits nLockTime = 0 with a final input sequence, so it is mineable at any time. Path B is
the CHECKLOCKTIMEVERIFY-gated backstop and its template (`refundHashB`) commits nLockTime = termEndLockTime
with a non-final input sequence, which is what CHECKLOCKTIMEVERIFY requires, so it is mineable only at or
after the term. A single shared hash would not work, because no one nLockTime can be both immediately
mineable for Path A and at-or-after-term for Path B. Both templates still bind inputCount = 1 and
inputIndex = 0, so neither path can add an input or redirect an output.

There is no path that pays anyone other than the funders. The Path A committee can only choose to refund
early. After the term, any party may broadcast Path B and the funds still go only to the funders. Path A
uses an M-of-N multisig rather than a single key so that no single party can force an early dissolve, and a
colluding committee can still only refund every funder, never redirect funds.

#### Committed parameters (byte-exact)
A validator recognizes this template by an EXACT, byte-for-byte canonical match, not a loose one. After
stripping the fixed-length committed-parameter prefix, the remaining bytes must match the canonical
two-branch structure exactly: `OP_IF <push-M> [N key pushes] <push-N> OP_CHECKMULTISIGVERIFY <push
refundHashA> OP_CHECKTEMPLATEVERIFY OP_ELSE <push termEndLockTime> OP_CHECKLOCKTIMEVERIFY OP_DROP <push
refundHashB> OP_CHECKTEMPLATEVERIFY OP_ENDIF`, with each push using its minimal encoding and no extra or
non-canonical operations. Any redeemScript that hashes to the collateral but deviates from this exact
pattern is not a recognized covenant and the registration is rejected. This closes the parser-confusion
risk where a crafted script could hash to the P2SH address yet hide dummy operations to fool recognition.
The validator then reads the committed values from the fixed prefix by position. The prefix (the five
registration parameters below, before OP_IF) is fixed-length, so it parses by position regardless of the
Path A committee size. The encodings are:

- `ownerKeyID`, a 20-byte CKeyID, pushed as `0x14` followed by 20 bytes.
- `votingKeyID`, a 20-byte CKeyID, pushed the same way.
- `operatorKey`, the 48-byte BLS operator public key (`pubKeyOperator`), pushed as `0x30` followed by 48
  bytes. Committing it means the funders approve who operates the node, and it is what makes a nonzero
  operator reward safe (see below). It fits well inside the 520-byte redeemScript limit.
- `payoutSharesHash`, 32 bytes, a single SHA-256 of the merged DIP-0026 payout-shares serialization (the
  `CMasternodePayoutShare` set from PR #7340), which is a CompactSize count followed by, for each share, a
  CompactSize scriptPayout length, the scriptPayout bytes, and the 2-byte little-endian reward. The exact
  serialization must track the deployed DIP-0026 type, since the share commitment has to match what the
  ProRegTx carries.
- `operatorReward`, the 2-byte little-endian nOperatorReward in basis points (10000 means 100 percent),
  pushed as a 2-byte value. The operator reward is paid to an address the operator sets later through a
  ProUpServTx, which is authorized by the operator key. Committing `operatorReward` alone bounds the size
  of the cut but not who receives it, so committing `operatorKey` as well is what closes the
  reward-capture channel, because the funders approve both the operator and the size of its reward at
  funding.
- The Path A committee, an M-of-N multisig of the form `<M> <controlPubKey_1> ... <controlPubKey_N> <N>
  OP_CHECKMULTISIGVERIFY`, each control public key a 33-byte compressed key. This is the one
  variable-length part of the script. It is NOT a registration parameter, so it is not equality-bound to
  the ProRegTx, and the recognized-template rule matches Path A as an M-of-N multisig within the exact
  canonical pattern above. N is bounded by the 520-byte redeemScript limit, not by MAX_PUBKEYS_PER_MULTISIG
  (20). The committed-parameter prefix is 132 bytes and the two-branch suffix is 81 bytes, so the constant
  footprint is 213 bytes, and each compressed committee key adds 34 bytes (a one-byte push opcode plus a
  33-byte key). The maximum is N = 9, since 213 + 9 times 34 = 519, within the 520-byte limit, while N = 10
  would need 553. The committee is a small set of funder representatives.
- `termEndLockTime`, a 4-byte little-endian value (the CHECKLOCKTIMEVERIFY operand).
- `refundHashA` and `refundHashB`, each a 32-byte StandardTemplateHash (the opcode's DefaultTemplateHash),
  one per branch. They commit the same refund outputs but differ in nLockTime and the input sequence, as
  described above.

These five registration parameters (owner key, voting key, operator key, payout-shares commitment,
operator reward) are the custody-critical fields. `CProRegTx::MakeSignString` (in the merged DIP-0026
implementation, PR #7340) already covers four of them explicitly (the payout list, operator reward, owner
key, voting key) and the operator key through the full payload hash it appends. The committed-parameter equality check plus the control
key's `MakeSignString` signature together reproduce, for covenant collateral, the authorization surface
that a P2PKH collateral key's signature already provides for ordinary external collateral.

#### Fan-out bounds (principal and rewards are bounded differently)
Two different limits apply, and conflating them was an error worth stating plainly.

- Principal refund. The refund template has one output per funder, an ordinary transaction output, so the
  funder count for PRINCIPAL is bounded only by the 100,000-byte standard transaction size
  (`src/policy/policy.h`). Hundreds of refund outputs stay far inside that limit.
- Reward split. Direct Layer 1 reward distribution uses the DIP-0026 payout shares, whose count is capped
  by consensus. The merged Dash Core implementation of DIP-0026 (PR #7340) rejects a payout set larger
  than 8 (`payouts.size() > 8`, "bad-protx-payouts-count"). So the number of funders who can receive
  rewards DIRECTLY on Layer 1 is bounded by that cap, which is the binding constraint, not the refund size.

For the MVP that pays rewards directly on Layer 1, the funder count is therefore capped at 8, the merged
DIP-0026 payout-share limit, with a minimum contribution to keep refund outputs well above dust. A pool
larger than 8 can still protect principal for any number of funders, but its rewards must be settled
through a Layer 2 accounting layer rather than split directly on Layer 1. The small direct-payout cap is
the main reason a retail-scale product needs the L2 accounting layer rather than direct on-chain reward
splitting.

### Funding (atomic)
Funders agree on a refund template T = [(addr_i, amount_i)] summing to the collateral minus the refund
fee, and on the registration parameters (the owner key, voting key, operator key, payout shares, and
operator reward).
They jointly construct one funding transaction, each funder's contribution as inputs and one output of
exactly the collateral amount paying to P2SH(redeemScript), where the redeemScript commits the two refund
hashes (refundHashA and refundHashB over the refund outputs T) and those registration parameters. Before signing (SIGHASH_ALL), each
funder independently verifies the full redeemScript preimage, that their own refund is correct, the
termEndLockTime, and the committed owner key, voting key, operator key, payout shares, and operator
reward. Verifying only the refund is not enough, because a coordinator bound to a correct refund could
otherwise register the node with their own owner key, or their own operator key against a nonzero operator
reward, and redirect future rewards. The funding
transaction is atomic, so if any funder refuses, no funds move, and paying to the P2SH hash is what binds
the funders to that exact set of committed parameters. The human-readable presentation of the committed
parameters is the "manifest", but its force comes from this hash commitment plus the consensus check at
registration, not from a signed side document.

### Registration (requires a DIP-0003 extension)
The masternode is registered against the covenant output as external collateral. Dash today rejects a
P2SH collateral for this purpose. For external collateral, the consensus code extracts a P2PKH key from
the collateral output and verifies the owner's message signature, the `MakeSignString` form, against it
(`src/evo/specialtxman.cpp` around the external-collateral path). A comment there states the extraction
works only for P2PK and P2PKH and fails for P2SH. A covenant address is P2SH and has no such single key,
so this DIP requires a DIP-0003 extension for covenant collateral with three parts.

1. Parameter binding. The ProRegTx supplies the covenant redeemScript. Consensus verifies that its
   HASH160 equals the collateral script hash, recognizes the covenant template, parses the committed
   owner key, voting key, operator key, payout-shares commitment, and operator reward, and requires the
   ProRegTx fields to equal them. The payout-shares check hashes the ProRegTx payoutShares with the same
   serialization used for the committed payoutSharesHash and compares the two. This prevents a coordinator
   from substituting their own owner key, their own operator key, or redirected shares, because those
   values are fixed by the redeemScript the funders committed to at funding.
2. Authority proof. Path A is an M-of-N committee, not a single key, so the registration signature is from
   any one of the committed committee keys (1-of-N). The extension generalizes the existing single-key
   `CheckStringSig` to accept a `MakeSignString` signature that verifies against any one of the N committee
   public keys parsed from Path A. Any committee member can register and no single member can block it.
   Since `MakeSignString` covers the full payload hash, this binds the operational fields that are not
   committed in the covenant, such as the service address, to whatever the registering member chose. A
   1-of-N signer is safe because part 1 has already equality-bound every custody-critical field, so the
   registrant cannot deviate them, and the one field they do set, the service address, is operator-fixable
   later through a ProUpServTx.
3. Persisted covenant marker. The redeemScript preimage is only on chain when the collateral is SPENT,
   not while it sits as a referenced UTXO, so a later ProUpRegTx validation that has only the collateral
   outpoint and its P2SH scriptPubKey cannot re-parse the covenant. To make the lifetime rule (see
   Operation) implementable, registration records a covenant-collateral marker in the deterministic
   masternode entry at the point where part 1 has already recognized the template and verified the
   committed parameters. Concretely, the marker is a single boolean on the masternode entry
   (`CDeterministicMN`, `src/evo/deterministicmns.h`, which already stores `collateralOutpoint`,
   `nOperatorReward`, and the state pointer). The committed registrar fields the rule depends on (owner
   key, voting key, operator key, payout shares, operator reward) are already in the masternode state, so
   no further committed data needs to be re-derived at ProUpRegTx time. ProUpRegTx validation reads the
   marker and rejects the transaction when it is set.

   Because the marker is consensus-critical, three implementation properties must hold, all stated here so
   they are not left to chance. First, the marker (`fCovenantCollateral`) is a SERIALIZED field of the
   masternode entry (`CDeterministicMN`), not a memory-only value, so it survives the snapshot-and-diff
   machinery that rebuilds the deterministic list across reorgs. It defaults to false for non-covenant and
   for pre-activation entries, with an activation-aware format-version migration. Second, the marker is
   IMMUTABLE after registration: it is set only when the entry is first constructed and is never changed
   by a later ProUpServTx, ProUpRegTx, ProUpRevTx, PoSe transition, or payment-state update. This matters
   because the list's diff machinery carries later changes as state diffs, so an entry-level field is only
   round-tripped correctly if it never changes after the initial add. Third, the marker is included in
   deterministic-list equality, snapshot verification, and any repair tooling, so a consensus-critical
   flag cannot silently diverge between nodes.

The funder authorization is therefore enforced by consensus at registration, derived from the atomic
funding transaction, not from any off-chain signed document. The committed-parameter layout and the
recognized-template rule are specified above and in the opcode DIP. A reference implementation and test
vectors remain to be written.

### Operation
Rewards are split to funders via DIP-0026 payout shares weighted by contribution. The operator runs the
node and, if the committed operator reward is nonzero, sets its own payout address through a ProUpServTx
(authorized by the operator key) and collects the committed reward fraction. That is the funder-approved
arrangement, since the funders committed both the operator key and the operator reward at funding.

The registrar fields (owner key, voting key, operator key, payout shares, operator reward) must not change
for the node's life, because a ProUpRegTx can otherwise rewrite them and is authorized by a single owner
key (`src/evo/specialtxman.cpp:1248`). A ProUpRegTx does not only change shares: it also rotates the
operator key, which resets the operator fields and PoSe-bans the node (`src/evo/specialtxman.cpp:395`),
and it rewrites the voting key. So freezing only the shares is not enough. For this fixed-membership MVP,
consensus rejects any ProUpRegTx against a masternode whose covenant marker is set (recorded at
registration, see part 3 above), which freezes every registrar field at the funder-committed values. ProUpServTx stays available, so the operator can keep the node in service and
receive its committed reward, but it cannot touch the custody-critical fields. This makes the owner key
inert for covenant collateral and neutralizes trust point 3 without threshold ECDSA.

The rule is expressed as "the registrar fields equal what the current covenant commits", not as a flat
"covenant nodes can never change" rule, so the dynamic-membership version (see Limitations) generalizes it
by committing a funder-quorum authority and updating the committed fields through a covenant replacement,
on the same consensus framework rather than a second incompatible one.

### Operator failure
Freezing every registrar field has a deliberate cost: a covenant node has no in-place operator
replacement. Installing a new operator key on Dash requires a ProUpRegTx, which this design rejects for
covenant collateral, so if the operator revokes (ProUpRevTx), has its key compromised, or simply
disappears and lets the node fall to a proof-of-service ban, the node cannot be re-keyed or revived where
it stands.

The recovery path is the cooperative early dissolve, Path A. Its branch carries no timelock, so it can be
exercised at any time to refund every funder, after which the funders reform a fresh pool with a new
operator. Principal is therefore not permanently lost to operator failure, though a timely early exit
depends on the committee reaching its signing threshold (the S5 row states the locked-until-term fallback
if it cannot). For this recovery to be a real guarantee rather than a new trust point, Path A MUST be an
M-of-N committee of funder representatives, not a single coordinator, so that no one party's absence can
block the early dissolve and no one party can force it. Path A is therefore an `OP_CHECKMULTISIGVERIFY`
over the committee keys (see the covenant script), which is the natural fit on Dash because the chain is
ECDSA-only, with no Schnorr or Taproot key aggregation. The committee size N is bounded by the 520-byte
redeemScript, which leaves room for about 9 committee keys once the committed parameters are accounted for,
so it is a small representative set rather than every funder. A colluding committee can
still only force a refund to all funders, never redirect funds, so a small committee is safe. An off-chain
threshold-ECDSA single key is the alternative if a larger committee or a rigid single-key script is wanted,
at the cost of multi-party-computation machinery. This version uses the on-chain multisig to keep the MVP
free of that machinery.

This choice has a cost beyond losing the node identity on reform. Dissolving spends the collateral and
destroys the masternode, so the reformed pool starts again at the back of the deterministic payment
rotation and pays the setup and transaction fees a second time. A negligent or malicious operator can
force repeated dissolves and erode the pool's compounded yield. None of this risks principal, because every
dissolve refunds the funders in full and the funders vet the operator key they commit at funding, but it is
a real efficiency cost and the strongest argument for the in-place operator replacement that the
dynamic-membership version adds (authorized by a funder quorum so an owner-appointed operator cannot
capture the reward).

### Refund
- Cooperative early dissolve: the Path A committee signs (M-of-N); the refund template pays all funders.
- Permissionless backstop: after termEndLockTime, any funder (or a watchtower) broadcasts Path B; the
  refund template pays all funders. This is the trustless guarantee.

### Fee strategy
The refund fee is fixed at funding, because the refund hashes commit the exact outputs, so the fee equals the
collateral minus the sum of the refund outputs and cannot be changed later. Dash has no Replace-By-Fee, so
a refund cannot be fee-bumped in place. The guarantee rests on the committed fee, and CPFP only helps at
the margin.

- The guarantee, a generously committed fee. Set the committed fee well above plausible future mempool
  minimums. The fee is negligible against a 1000 DASH collateral, so overpaying by a wide margin costs
  almost nothing, and it is the only mechanism that actually guarantees the refund is acceptable to the
  mempool at exit time.
- Acceleration only, reactive Child-Pays-For-Parent. If the committed fee is at or above the floor at exit
  time, the refund is already acceptable, and a funder who wants it confirmed faster can attach a high-fee
  child to their own refund output. Dash mines by ancestor feerate (`src/node/miner.cpp` selects packages
  by ancestor score), so a high-fee child raises the parent's mining priority. Each funder can do this
  independently, with no coordination.

CPFP does NOT rescue a refund whose committed fee is below the floor. The mempool minimum feerate is a
rolling floor, and Dash rejects an individual transaction below it except inside a package
(`src/validation.cpp`). If the committed fee is below the floor at exit time, the refund never enters any
mempool, so there is nothing for a child to attach to, and Dash has no usable package-relay path on
mainnet (`submitpackage` is regtest-only, `src/rpc/mempool.cpp`). The residual risk is therefore a
liveness delay, not a loss: under extreme sustained congestion a too-low refund waits until the rolling
floor recedes (Dash floors decay over time), after which it confirms. Principal is never lost, because
Path B stays permissionless and the covenant only ever pays the funders. The mitigation is to commit the
fee generously in the first place.

The child MUST be built reactively, after the funder observes the actual refund transaction in the
mempool, not prebuilt. The refund transaction ID is malleable, because Dash has no SegWit and the
template deliberately does not commit the spending scriptSig (see the opcode DIP), so a prebuilt child
referencing a guessed parent txid can be orphaned by an alternate-scriptSig broadcast of the same refund.
A child built against the observed parent txid is not affected. Output dust and standardness are not a
concern here, because the minimum contribution (10 DASH) keeps every refund output far above the dust
threshold and the single-input, committed-output refund is a standard transaction.

## Trust analysis (against the security requirements in the Informational DIP)
- S1 (no single party can spend the collateral to themselves): holds by construction ONCE the opcode
  hash and the covenant script forms are pinned. Every spend path is covenant-forced to the funder
  template.
- S2 (no single party can redirect rewards): holds for the MVP ONCE the covenant-bound authority rules
  are specified and implemented, the registration parameter-binding (owner key, voting key, operator key,
  payout shares, operator reward) plus the rule that consensus rejects any ProUpRegTx against covenant
  collateral. Committing the operator key closes the operator-reward channel, since the operator and its
  reward are funder-approved. Not met by the refund covenant alone. Until the authority rules are pinned
  and reviewed, treat it as open.
- S3 (every funder can reclaim principal without cooperation): the registration extension, the fee
  strategy, and the opcode hash are now specified, so the design path holds, pending a reference
  implementation and review. Path B is permissionless after the term. One residual: under extreme
  sustained congestion a refund whose committed fee fell below the mempool floor is delayed until the
  floor recedes (see Fee strategy), so the reclaim is eventual, not necessarily prompt. Principal is not
  at risk.
- S4 (reorg/mempool safety): to be proven; depends on the opcode DIP's analysis.
- S5 (one bad actor cannot destroy the node): before termEndLockTime, no party can spend the principal or
  redirect rewards, because the registrar fields are frozen (ProUpRegTx rejected) and principal is
  covenant-locked. The residual is liveness, not theft. Because the frozen registrar also blocks in-place
  operator replacement, a revoked or absent operator cannot be swapped out where the node stands (see
  Operator failure). The recourse is the M-of-N committee Path A early dissolve, which refunds every funder
  at any time, after which they reform with a new operator, so principal is never trapped. This depends on
  the Path A committee reaching its threshold. If the committee cannot reach M signatures during an operator
  failure, the early dissolve cannot fire and capital stays locked until termEndLockTime, after which Path B
  refunds everyone permissionlessly. So the worst case is a delay to the term, never a loss. After the term,
  any party can broadcast Path B and dissolve the node, which is by design. Indefinite continuity needs a
  renewal protocol, either cooperative quorum replacement or bounded nesting (opcode DIP, Recursion).

## Rationale

The covenant approach is preferred over native multi-funder collateral (Option 2 in the Informational
DIP) because it removes funders as signers, breaking the 15-key on-chain multisig wall, and reuses a
general-purpose primitive rather than bespoke masternode-system validation. Option 2 is documented as
the alternative.

Fixed-term and fixed-membership comes first because it is the version that is fully trustless for
principal without recursive covenants, which are the contested, highest-risk part of the design.
Shipping the safe MVP first is deliberate.

## Backwards compatibility
New masternodes using covenant collateral require the opcode DIP and the registration extension to be
active. Existing single-owner masternodes are unaffected. Activation via the same coordinated network
upgrade as the opcode DIP.

## Limitations and future work
- Fixed term. After termEndLockTime the backstop can dissolve a healthy node. Keeping a node alive
  indefinitely requires renewal, re-committing with a later term, which can be done either by cooperative
  quorum replacement (no recursion rule, but it needs a live quorum at each renewal) or by bounded,
  pre-committed nesting if the opcode allows it. See the opcode DIP's Recursion section, where the choice
  is left open.
- No individual exit, and no member swap, in the MVP. A swap is two reassignments, the departing funder's
  principal claim (in the covenant refund template) and their reward share (in the DIP-0026 payout
  shares). The reward half is already covered by the covenant-bound owner authority above. The principal
  half always touches the collateral UTXO, so it needs a covenant replacement (cooperative quorum
  replacement, which needs no recursion rule, or bounded nesting if allowed) plus atomic outpoint
  replacement to avoid tearing down the node. The MVP defers both, and the covenant-bound authority is the
  seam through which the dynamic version extends.
- No in-place operator replacement. Freezing the registrar to stop reward and vote capture also blocks the
  only path to re-key a masternode, so a revoked or absent operator is handled by the M-of-N committee
  Path A early dissolve and a reform, not by swapping the operator in place (see Operator failure). This
  makes an on-chain multisig committee for Path A a requirement, and the reform costs the node identity,
  the place in the deterministic payment rotation, and a second round of setup and fees. A negligent
  operator can grief by forcing repeated dissolves, bounded to forced refunds since no dissolve can take
  principal. In-place operator replacement under a funder quorum is deferred to the dynamic version.
- Fixed refund fee, committed at funding. Dash has no Replace-By-Fee. The guarantee is a generously
  committed fee (see Fee strategy). CPFP can only accelerate an already-acceptable refund, not rescue one
  whose fee is below the rolling mempool floor, since Dash has no usable package relay. The residual is a
  liveness delay under extreme sustained congestion, with principal never at risk.
- Retail scale and churn require a Dash Platform accounting layer on top. This DIP is the L1 custody
  piece only.

## Open design questions
- A reference implementation and test vectors for the committed-parameter parser and the DIP-0003
  validation. The byte-exact layout is specified under Specification, and the authority model is settled
  (covenant-bound parameter binding at registration, a persisted covenant marker, and a rule that rejects
  any ProUpRegTx against a marked masternode for the MVP). What remains is the implementation and its
  test vectors.
- The covenant-bound ProUpRegTx rule for the dynamic version, the funder-quorum authority and how a
  covenant replacement updates the committed shares atomically with the principal claims.
- Recursive covenants for dynamic membership and individual exit, versus periodic dissolve-and-reform
  coordinated on Platform. The MVP authority model is built to extend into the recursive case, so this
  choice and the recursion stance should be decided together.
- Reorg/replay safety of the refund spend and its interaction with the masternode-removal rule. (The fee
  strategy is now specified under Specification, so it is no longer an open item.)

## Acknowledgements
The covenant direction originated with pshenmic. Thanks also to Michael (kxcd). This builds on
DIP-0003, DIP-0026, and the Output-Template Covenant Opcode DIP.

## Copyright
This document is licensed under the MIT License.
