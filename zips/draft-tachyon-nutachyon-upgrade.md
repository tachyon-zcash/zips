```
ZIP: XXX
Title: Deployment of the NuTachyon Network Upgrade
Owners: Connor O'Hara <connor@s1nus.com>
Status: Draft
Category: Consensus / Network
Created: 2026-09-23
License: MIT
```


# Terminology

The key words "MUST" and "MUST NOT" in this document are to be
interpreted as described in BCP 14 [^BCP14] when, and only when, they appear in
all capitals.

The term "network upgrade" is defined in ZIP 200.[^zip-0200] "Testnet" and
"Mainnet" are defined in the Zcash Protocol Specification.[^protocol-networks]


# Abstract

This ZIP defines the deployment of NuTachyon, activating the Tachyon shielded
pool, its bundle format, and consensus-level proof aggregation.


# Motivation

Tachyon introduces a new shielded pool with recursive proofs and prunable
nullifier state. Deployment requires coordinated changes to consensus validation,
chain-state accounting, transaction encoding, and peer compatibility.


# Privacy Implications

NuTachyon introduces a separate shielded pool; activation does not migrate
existing notes into it. See the shielded-protocol ZIP's
[Privacy and Security Implications](draft-tachyon-shielded-protocol.md#privacyandsecurityimplications)
and the bundle-format ZIP's
[Privacy Implications](draft-tachyon-bundle-format.md#privacyimplications).


# Non-requirements

This ZIP does not specify P2P aggregation messages, covered-transaction discovery,
or mempool and relay policy.


# Specification

## NuTachyon deployment

Under ZIP 200's activation mechanism,[^zip-0200] nodes MUST apply the NuTachyon
rules starting with the block at `ACTIVATION_HEIGHT (NuTachyon)`. The preceding
block remains subject to the pre-upgrade rules.

NuTachyon activates the following specifications:

- [Tachyon Shielded Protocol](draft-tachyon-shielded-protocol.md#specification):
  cryptographic constructions, proof statements, epochs, and pool-state rules.
- [Tachyon Bundle / Aggregate Transaction Format](draft-tachyon-bundle-format.md#specification):
  bundle encodings, transaction-digest contributions, and bundle- and block-scoped
  validity rules.

Nodes MUST enforce both specifications.


## Network upgrade constants

Deployment constants are unassigned. The
[PoC's](#referenceimplementation) experimental values are not deployment assignments.

| Constant | Testnet | Mainnet |
| --- | --- | --- |
| `CONSENSUS_BRANCH_ID (NuTachyon)` | TBD | TBD |
| `ACTIVATION_HEIGHT (NuTachyon)` | TBD | TBD |
| `MIN_NETWORK_PROTOCOL_VERSION (NuTachyon)` | TBD | TBD |

The consensus branch ID is a globally unique, nonzero 32-bit unsigned integer,
as specified in ZIP 200.[^zip-0200] Activation heights are nonzero 32-bit unsigned
integers. Minimum network protocol versions are positive 32-bit signed integers.


## Network upgrade changes

### Tachyon pool initialization

The Tachyon pool is empty before the activation block. From activation onward,
its pool height is a 32-bit unsigned integer equal to block height minus
activation height. The activation block has pool height zero and begins epoch zero. Epoch boundaries
and accumulator updates follow [Epochs](draft-tachyon-shielded-protocol.md#epochs) and
[Tachygram accumulator](draft-tachyon-shielded-protocol.md#tachygramaccumulator).

TODO: Finalize the epoch length, initial anchor, and its acceptance at activation
in the referenced protocol specification.


### Transaction format and digests

The choice between ZIP 248's extensible transaction format[^zip-0248] and a new
transaction format remains open. No transaction version or version group ID is
assigned here.

The bundle-format ZIP specifies the
[Transaction digest contributions](draft-tachyon-bundle-format.md#transactiondigestcontributions),
preserving the separation between effecting and authorizing data used by
ZIP 244.[^zip-0244]

TODO: Specify the enclosing transaction format, Tachyon-specific digest algorithms
and signature-hash integration, and any corresponding updates to transaction
relay under ZIP 239.[^zip-0239]


### Chain value pool balance

The Tachyon chain value pool balance, an integer number of zatoshis, is the
negated sum of all `valueBalanceTachyon` fields in the block chain, and is zero
before activation. See
[Value balance and the binding signature](draft-tachyon-bundle-format.md#valuebalanceandthebindingsignature).

Nodes MUST apply ZIP 209's non-negativity and total-supply checks with the Tachyon
pool included.[^zip-0209] Proof aggregation does not merge value balances: each
bundle's value balance contributes once, whether it carries a proof or a pointer.


### Chain history commitment

NuTachyon extends the ZIP 221 chain history commitment.[^zip-0221] The PoC adds
history-tree fields for the earliest and latest Tachyon pool-state anchors and
a count of transactions containing Tachyon bundles.[^poc-history]

TODO: Specify the history-tree encoding and commitment update, including
activation-boundary behavior, in ZIP 221 and the Zcash Protocol Specification.


## Backward compatibility

Before activation, upgraded and pre-NuTachyon nodes apply the same consensus
rules and can remain connected. Nodes MUST NOT accept Tachyon bundles in blocks
before activation. After activation, pre-NuTachyon software cannot validate the
new consensus rules.

This ZIP does not require closing existing shielded pools.

TODO: Specify which transaction versions remain valid after activation and the
coinbase rules for the chosen format.


# Reference implementation

[Zakura PR #795](https://github.com/zakura-core/zakura/pull/795) implements a
NuTachyon proof of concept (PoC) behind `zcash_unstable = "nutachyon"`. This ZIP
references [revision `ec58f4bb`](https://github.com/zakura-core/zakura/commit/ec58f4bb797d2ad0390a157e175bc94c09ad979e).

The PoC uses v7 transactions; this does not settle the ZIP 248 decision. It
schedules neither public Testnet nor Mainnet activation.

It implements Tachyon proof and signature verification, epoch-aware state,
value-pool accounting, history commitments, and miner-side proof aggregation.
Its mempool accepts Tachyon transactions whose proofs cover only their own
actions; P2P aggregation and relay remain work in progress. These policies are
not consensus requirements.


# References

[^BCP14]: [Information on BCP 14 — RFC 2119 and RFC 8174](https://www.rfc-editor.org/info/bcp14)

[^protocol-networks]: [Zcash Protocol Specification: Mainnet and Testnet](protocol/protocol.pdf#networks)

[^zip-0200]: [ZIP 200: Network Upgrade Mechanism](zip-0200.rst)

[^zip-0209]: [ZIP 209: Prohibit Out-of-Range Chain Value Pool Balances](zip-0209.rst)

[^zip-0221]: [ZIP 221: FlyClient — Consensus-Layer Changes](zip-0221.rst)

[^zip-0239]: [ZIP 239: Relay of Version 5 Transactions](zip-0239.rst)

[^zip-0244]: [ZIP 244: Transaction Identifier Non-Malleability](zip-0244.rst)

[^zip-0248]: [ZIP 248: Extensible Transaction Format](zip-0248.rst)

[^poc-history]: [Zakura PoC: V4 history-tree node data](https://github.com/zakura-core/zakura/blob/ec58f4bb797d2ad0390a157e175bc94c09ad979e/crates/zakura-mmr-tree/src/node_data.rs#L300-L344)
