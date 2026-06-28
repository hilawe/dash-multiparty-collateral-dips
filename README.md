# Dash Multi-Party Collateral DIPs

Draft Dash Improvement Proposals (DIPs) for non-custodial, multi-party masternode collateral. A Dash
masternode requires its collateral to be a single output controlled by one party, which makes any
pooled or shared masternode custodial. DIP-0026 (multi-party payouts) made reward distribution
non-custodial, and this work addresses the unsolved half, the collateral itself.

These are working drafts shared for review. Nothing here has been submitted to the official
dashpay/dips repository yet.

## Start here

Read the Informational DIP first for the problem statement, the trust model, and the design space, then
the two Standard DIPs in the order they are listed below. The open questions at the end of each Standard
DIP are where directional feedback is most useful. These are early drafts offered for guidance, not
finished specifications, and input on whether the overall direction is the right one is more valuable than
line-level detail at this stage.

## Contents

- [dip-info-mn-collateral.md](dip-info-mn-collateral.md) (Informational): the problem statement, the
  trust model, the requirements, and the design space, with a recommendation. Start here.
- [dip-std-covenant-opcode.md](dip-std-covenant-opcode.md) (Standard): a general-purpose covenant
  opcode (OP_CHECKTEMPLATEVERIFY, a working name) that lets an output commit to an exact spending
  template. Early draft.
- [dip-std-shared-collateral.md](dip-std-shared-collateral.md) (Standard): the shared-collateral
  application that uses the covenant opcode. Early draft.

## Status

The Informational DIP is the most developed. The two Standard DIPs are concrete proposals with their
open items flagged, not finished specifications. DIP numbers are not yet assigned, and the editors
would assign them on formal submission.

## How to review and contribute

Feedback is welcome as issues and pull requests. The most useful input right now is directional. Is
the approach sound, and is the recommendation (a covenant opcode as the foundation, with a fixed-term
non-recursive first version) the right call? Specific open questions are listed at the end of each
Standard DIP.

## Authors

Hilawe Semunegus (hilawe). Special thanks to pshenmic and Michael (kxcd).

## License

MIT. See [LICENSE](LICENSE).
