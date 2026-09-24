```
ZIP: XXX
Title: Tachyon Shielded Protocol
Owners: Connor O'Hara <connor@s1nus.com>
Status: Draft
Category: Consensus
Created: 2026-09-23
License: MIT
```


# Terminology

The key word "MUST" in this document is to be interpreted as described in
BCP 14 [^BCP14] when, and only when, it appears in all capitals.


# Abstract

This document proposes the Tachyon shielded protocol, which defines a new shielded pool with various scalability advantages. Tachyon enables validators to prune nullifier state, and supports aggregation of ZK proofs.

# Motivation

The Orchard shielded protocol specified by ZIP-224 introduced a proof system designed around a curve cycle, with two main motivations:

- Removing the system's reliance on a "trusted setup", both to harden the trust assumptions of the shielded protocol and to remove the coordination complexity of the setup process.
- Introduce the possibility of integrating known scaling techniques based on recursive zero-knowledge proofs in a future upgrade.

While the first goal was achieved, the Orchard shielded protocol does not utilize Halo 2's recursion capability for a scalable protocol.

We are thus motivated to deploy a new shielded protocol focused on scalability, fully utilizing recursive proofs to overcome two major bottlenecks:

- Nullifier state growth:
    - All of Zcash's existing shielded protocols have an unfortunate drawback of infinitely growing nullifier state. All validators must store the full historical nullifier set, the entirety of which must be retained to validate shielded transactions.
    - This limits scalability, as raising the transaction throughput would result in linear growth of disk and memory requirements for validators.
- Proof size and verification cost:
    - Orchard uses a single proof per bundle, but its size grows linearly with the number
      of actions. Proofs from separate transactions cannot be recursively combined under
      the deployed protocol.[^zip-0225]
    - Increasing transaction throughput therefore increases the proof data that validators
      must receive and store, as well as their total proof-verification workload.
    
Tachyon remediates both bottlenecks by leveraging recursive proofs:
- By requiring users to privately obtain and supply a recursive zero-knowledge proof of their notes spendability along with their transaction, nullifier state may be organized into [_epochs_](#epochs) and periodically pruned by validators.
- By allowing proofs from multiple transactions to be aggregated into a single proof, storage and verification costs can be amortized for proof data.

# Specification

The Tachyon protocol MUST be implemented as specified in the Zcash Protocol
Specification.[^protocol-tachyon]


## Curves and fields

Tachyon retains the Pallas / Vesta (Pasta) curve cycle and finite fields used by
Orchard, as described in ZIP 224.[^zip-0224] The base field of each curve is the
scalar field of the other; their parameters are unchanged.[^protocol-pallasandvesta]

Pallas remains the application curve, used for RedPallas signatures and value
commitments. Unlike Orchard's non-recursive use of Halo 2, Tachyon's Ragu proving
system utilizes the full curve cycle for recursive proof composition.[^ragu]

## Proving system

Tachyon uses the Ragu proving system,[^ragu] with R1CS-like arithmetization inspired by Bootle16[^bootle16] and efficient recursion via split accumulation.[^bclms21]

## Stamp

A stamp packages a recursive zero-knowledge proof, a pool-state anchor, and the
[tachygrams](#tachygrams) associated with the actions it covers. The anchor
identifies the pool state referenced by the proof. The proof binds the covered
actions to the tachygrams and establishes the corresponding note-creation or
spendability statements.[^protocol-tachyon-stamp-verification]

A stamp can cover actions from a single transaction or aggregate actions from
multiple transactions. [Aggregation](draft-tachyon-aggregation-protocol.md) replaces
the input stamps with a single stamp, while the actions and their signatures
remain in their respective transactions.[^tachyon-proof-tree]


## Epochs

An epoch is a fixed-length interval of consecutive blocks, identified by an index
derived from block height. Each note has an epoch-specific nullifier. A spend
publishes nullifiers for the epoch of its referenced pool state and the following
epoch, allowing inclusion across an epoch boundary.[^tachyon-nullifiers]

When validating a transaction, validators check for duplicate nullifiers in the
current and immediately preceding epochs. Recursive spendability proofs cover
earlier history, allowing validators to prune older nullifier state from their
active duplicate-checking set.
[^tachyon-tachygrams] [^tachyon-proof-tree]

Epoch boundaries and cross-epoch validation rules MUST be implemented as specified
in the Zcash Protocol Specification.[^protocol-tachyon-epochs]


## Proof tree

Instead of Orchard's single action circuit, Tachyon uses a recursive proof tree of
specialized proof-carrying data (PCD) steps. Each step checks its own constraints
and combines up to two input PCDs into a new PCD.

Spend proofs combine evidence of note creation, correct per-epoch nullifier
derivation, and continued unspentness up to an anchor. Separate output steps prove
correct construction of new notes. Both branches produce stamp proofs whose
headers bind an anchor and commitments to the action-digest and tachygram
multisets. Tachygrams comprise nullifiers, note commitments, and padding values.

Stamps at a common anchor can be merged recursively, first to cover a transaction's
actions and then to aggregate independently constructed transactions. Anchor-chain
proofs allow stamps to be advanced to a common later anchor before merging.
Validators check the resulting proof against commitments reconstructed from the
covered actions and published tachygrams, and check the anchor against chain state.

- PCD step statements and composition rules: [^protocol-tachyon-proof-tree]
- Stamp public inputs and verification: [^protocol-tachyon-stamp-verification]
- Design description: [^tachyon-proof-tree]

## Commitment schemes

Tachyon retains Orchard's homomorphic Pedersen value commitments on Pallas,
including the value and randomness generators. A Tachyon action commits to a
signed value, positive for spends and negative for outputs.

For note commitments, Tachyon uses a domain-separated Poseidon sponge over the
Pallas base field in place of Orchard's Sinsemilla construction. It commits to the
recipient's payment key, value, and nullifier trapdoor using note-commitment
randomness, producing a field element.

Tachyon additionally uses deterministic Pedersen polynomial commitments on Vesta
for action-digest and tachygram multisets. Each multiset is encoded as the roots of
a monic polynomial, whose coefficients are committed using Ragu's fixed generators.
These commitments preserve multiplicity but not order, and have no blinding term.

- Value commitment scheme: [^protocol-valuecommit]
- Note commitment construction: [^protocol-tachyon-notecommit]
- Multiset commitment construction: [^protocol-tachyon-multisetcommit]
- Design and implementation: [^tachyon-notes] [^tachyon-authorization] [^tachyon-multisetcommit]


## Tachygrams

A tachygram is a Pallas base-field element representing a note commitment, a
nullifier, or a padding value. A spend contributes nullifiers for its epoch and
the next; an output contributes a note commitment and a padding tachygram. Both
action types contribute exactly two values, so the tachygram count alone does not
reveal the split between spends and outputs.

Tachygrams are published as an untyped multiset. The zero-knowledge proof enforces
their correct derivation and association with the covered actions.
Consensus therefore need not distinguish note commitments from nullifiers or
padding: it verifies the proof and applies the same duplicate-rejection
rules and accumulator updates to all tachygrams.[^tachyon-tachygrams]


## Tachygram accumulator

Tachyon replaces Orchard's note commitment tree with an accumulator over
tachygrams: note commitments, nullifiers, and padding values. Each stamp's
tachygram-multiset commitment is absorbed into a Poseidon hash chain, which also
records epoch transitions, to derive pool-state anchors.

The tachygram accumulator MUST be implemented as specified in the Zcash Protocol
Specification.[^protocol-tachyon-accumulator]


## Keys and addresses

Tachyon's key design deliberately stays close to Orchard. It retains the 32-byte
spending key $\mathsf{sk}$, derivation of the spend-authorizing key $\mathsf{ask}$
and base-field nullifier key $\mathsf{nk}$, and RedPallas spend authorization with
a spend-validating key $\mathsf{ak}$ and randomized per-spend keys.

The principal changes are:

- The proof-authorizing key is explicitly represented as
  $\mathsf{pak} = (\mathsf{ak}, \mathsf{nk})$, allowing proof construction without
  the spend-authorizing key.
- The recipient is represented by a field-element payment key $\mathsf{pk}$,
  derived from $\mathsf{ak}$ and $\mathsf{nk}$ using domain-separated Poseidon,
  in place of Orchard's diversifier and diversified transmission key.
- Orchard's viewing-key and diversified-address machinery is omitted from the
  core protocol. Address management and payment-data exchange are handled by
  higher-level, out-of-band protocols.

Key components and derivation: [^protocol-tachyon-keys].
Design and implementation: [^tachyon-keys] [^tachyon-key-derivation].


## Signature schemes


## Notes


## Note commitments


## Nullifiers


## Anchors


## Actions


### Spend actions


### Output actions


## Authorization


## Value balance


## Proof statements


### Spend action statement


### Output action statement


## Consensus rules


# Privacy Implications


## Observable protocol data


## Linkability


# Requirements


# Non-requirements


# Rationale


# Security Implications


## Soundness


## Nullifier security


# Test vectors


# Deployment


# Reference implementation


# Open issues


# References

[^BCP14]: [Information on BCP 14 — RFC 2119 and RFC 8174](https://www.rfc-editor.org/info/bcp14)

[^protocol-tachyon]: Zcash Protocol Specification, Tachyon protocol. TODO: Add the version and section references once the Tachyon specification is written.

[^protocol-tachyon-epochs]: Zcash Protocol Specification, Tachyon epoch boundaries and cross-epoch validation rules. TODO: Add the version and section references once specified.

[^protocol-tachyon-proof-tree]: Zcash Protocol Specification, Tachyon PCD step statements and composition rules. TODO: Add the version and section references once specified.

[^protocol-tachyon-stamp-verification]: Zcash Protocol Specification, Tachyon stamp public inputs and proof verification. TODO: Add the version and section references once specified.

[^protocol-valuecommit]: [Zcash Protocol Specification: Homomorphic Pedersen commitments (Sapling and Orchard)](protocol/protocol.pdf#concretehomomorphiccommit)

[^protocol-tachyon-notecommit]: Zcash Protocol Specification, Tachyon note commitment construction. TODO: Add the version and section references once specified.

[^protocol-tachyon-multisetcommit]: Zcash Protocol Specification, Tachyon multiset commitment construction. TODO: Add the version and section references once specified.

[^protocol-tachyon-accumulator]: Zcash Protocol Specification, Tachygram accumulator and pool-state anchor updates. TODO: Add the version and section references once specified.

[^protocol-tachyon-keys]: Zcash Protocol Specification, Tachyon key components and derivation. TODO: Add the version and section references once specified.

[^tachyon-keys]: [The Tachyon Book: Key Hierarchy](https://github.com/tachyon-zcash/tachyon/blob/9abdcec1a98f96d91e612b3a43f5a4140de989f0/book/src/keys.md)

[^tachyon-key-derivation]: [Tachyon reference implementation: Key derivation](https://github.com/tachyon-zcash/tachyon/blob/9abdcec1a98f96d91e612b3a43f5a4140de989f0/crates/tachyon/src/keys/private.rs)

[^tachyon-tachygrams]: [The Tachyon Book: Tachygrams](https://github.com/tachyon-zcash/tachyon/blob/9abdcec1a98f96d91e612b3a43f5a4140de989f0/book/src/tachygrams.md)

[^tachyon-nullifiers]: [The Tachyon Book: Nullifiers](https://github.com/tachyon-zcash/tachyon/blob/9abdcec1a98f96d91e612b3a43f5a4140de989f0/book/src/nullifiers.md)

[^tachyon-notes]: [The Tachyon Book: Notes](https://github.com/tachyon-zcash/tachyon/blob/9abdcec1a98f96d91e612b3a43f5a4140de989f0/book/src/notes.md)

[^tachyon-authorization]: [The Tachyon Book: Authorization — Value Balance](https://github.com/tachyon-zcash/tachyon/blob/9abdcec1a98f96d91e612b3a43f5a4140de989f0/book/src/authorization.md#value-balance)

[^tachyon-multisetcommit]: [Tachyon reference implementation: Multiset commitments](https://github.com/tachyon-zcash/tachyon/blob/9abdcec1a98f96d91e612b3a43f5a4140de989f0/crates/tachyon/src/primitives/sets.rs)

[^tachyon-proof-tree]: [The Tachyon Book: Proof tree](https://github.com/tachyon-zcash/tachyon/blob/9abdcec1a98f96d91e612b3a43f5a4140de989f0/book/src/proof-tree.md)

[^zip-0224]: [ZIP 224: Orchard Shielded Protocol](zip-0224.rst)

[^protocol-pallasandvesta]: [Zcash Protocol Specification: Pallas and Vesta](protocol/protocol.pdf#pallasandvesta)

[^zip-0225]: [ZIP 225: Version 5 Transaction Format](zip-0225.rst)

[^ragu]: [Ragu proving system](https://tachyon.z.cash/ragu/)

[^bootle16]: Jonathan Bootle, Andrea Cerulli, Pyrros Chaidos, Jens Groth, and Christophe Petit. [Efficient Zero-Knowledge Arguments for Arithmetic Circuits in the Discrete Log Setting](https://eprint.iacr.org/2016/263). Cryptology ePrint Archive, Paper 2016/263, 2016.

[^bclms21]: Benedikt Bünz, Alessandro Chiesa, William Lin, Pratyush Mishra, and Nicholas Spooner. [Proof-Carrying Data without Succinct Arguments](https://eprint.iacr.org/2020/1618). Cryptology ePrint Archive, Paper 2020/1618, 2021.
