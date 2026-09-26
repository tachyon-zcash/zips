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
Tachyon decouples note-encryption and transmission semantics from the shielded
protocol, allowing more-scalable payment protocols to be built on top.

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


Tachyon decouples note transmission from the shielded protocol. The blockchain can still carry arbitrary payloads, but the Tachyon shielded protocol treats them as opaque rather than prescribing their encryption or note-transmission semantics. Higher-level payment protocols define how recipients discover and decrypt this data, and may use private information retrieval (PIR) to retrieve it privately.[^tachyon-secret-distribution] [^tachyon-payment-protocols]

Encryption, transport, and discovery are not prescribed by this
ZIP. This separation permits note transmission and discovery to evolve
independently of consensus, including approaches that avoid scanning every
transaction. [^tachyon-payment-protocols]

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

Tachyon uses the Ragu proving system,[^ragu] with R1CS-like arithmetization inspired by Bootle16[^bootle16], efficient recursion via split accumulation, and its online polynomial oracle and transcript bridging features.[^bclms21]

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
the next; an output contributes a note commitment and a padding tachygram to the stamp's tachygram multiset. Both
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
  core protocol. Address management, viewing capabilities, transmission keys,
  note encryption, and retrieval are instead defined by higher-level payment
  protocols. Encrypted notes can still be published as opaque on-chain
  payloads.[^tachyon-payment-protocols]

Key components and derivation: [^protocol-tachyon-keys].
Design and implementation: [^tachyon-keys] [^tachyon-key-derivation].


## Notes and note commitments

A Tachyon note has the form $(\mathsf{pk}, v, \psi, \mathsf{rcm})$, where
$\mathsf{pk}$ is the recipient's payment key, $v$ is a non-negative value in
zatoshis, $\psi$ is the nullifier trapdoor, and $\mathsf{rcm}$ is note-commitment
randomness. The payment key, nullifier trapdoor, and commitment randomness are Pallas base-field elements. The value is a non-negative integer amount in zatoshis, also encoded as a field element when computing the commitment. Unlike
Orchard, the note has no $\rho$ field linking it to the nullifier of a spend in
the same action.

The note commitment is a field element computed using domain-separated Poseidon
in place of Orchard's Sinsemilla construction:

$$\mathsf{cm} = \mathsf{Poseidon}_{\texttt{Tachyon-CmDerive}}(\mathsf{rcm}, \mathsf{pk}, v, \psi).$$

An output action publishes $\mathsf{cm}$ together with a padding tachygram. [^tachyon-note-implementation] [^tachyon-tachygrams]
The note structure and commitment MUST be implemented as specified in the Zcash
Protocol Specification.[^protocol-tachyon-notecommit]


## Nullifiers

Unlike Orchard's fixed nullifier per note, Tachyon uses _evolving nullifiers_,
deriving a different nullifier for each [epoch](#epochs).[^evolving-nullifiers]
A per-note master key $\mathsf{mk}$ is derived from the note's $\psi$ and the
recipient's nullifier key $\mathsf{nk}$. Abstractly:

$$\mathsf{mk} = \mathsf{Poseidon}_{\texttt{Tachyon-NfMaster}}(\psi, \mathsf{nk}),$$

$$\mathsf{nf}_e = \mathsf{PRF}^{\mathsf{nfTachyon}}_{\mathsf{mk}}(e).$$

The PRF uses a domain-separated Poseidon sponge to derive groups of consecutive
epochs' nullifiers. Its outputs are Pallas base-field elements.[^tachyon-nullifiers]

A spend publishes $\mathsf{nf}_e$ and $\mathsf{nf}_{e+1}$, where $e$ is the epoch
of its referenced pool state. The proof establishes that the note's applicable
nullifiers were absent between its creation and that state; validators perform
the recent duplicate checks described under [Epochs](#epochs). This combines
proofs of historical unspentness with a prunable consensus nullifier
set.[^tachyon-proof-tree]

Nullifier derivation and its proof constraints MUST be implemented as specified
in the Zcash Protocol Specification.[^protocol-tachyon-nullifiers]


## Consensus rules

Tachyon consensus validation MUST be implemented as specified in the Zcash
Protocol Specification.[^protocol-tachyon-consensus] The principal checks are:

- Bundle fields have canonical encodings and satisfy their type and range
  constraints. Each action's value commitment and authorization key are
  non-identity Pallas points. A bundle with no actions has zero value balance.
- Every action signature and each bundle's binding signature verify over the
  containing transaction's signature hash. The binding signature enforces
  consistency between the action value commitments and the declared value
  balance. These checks remain per-transaction after aggregation.
- Each stamp's declared coverage matches the actions in its own bundle and all
  bundles referring to it. Those references resolve to a proof-bearing bundle in
  the same block, the covered action descriptors are distinct, and the stamp
  publishes exactly two tachygrams per covered action.
- Each stamp proof verifies against its anchor and the multiset commitments
  reconstructed from the covered action digests and published tachygrams.
- Each stamp anchor identifies an accepted end-of-block pool state in the
  epoch of the block being validated or the immediately preceding epoch.
- All tachygrams in a block are distinct, and none repeats a tachygram published
  in an earlier block of the current or immediately preceding epoch. This check
  treats note commitments, nullifiers, and padding identically.
- The pool-state accumulator advances through the block's proof stamps in
  transaction order, incorporating epoch transitions as specified under
  [Tachygram accumulator](#tachygramaccumulator).

Opaque payment-protocol payloads remain subject to bundle-format encoding rules,
and their bytes are committed by the transaction's signature hash.[^tachyon-bundle-payload]

Wire-format details and cross-transaction coverage are specified by the
[bundle-format](draft-tachyon-bundle-format.md) and
[aggregation](draft-tachyon-aggregation-protocol.md) ZIPs.


# Privacy and Security Implications

Confidential note transmission, recipient authentication, and note discovery are
responsibilities of the chosen payment protocol. On-chain payloads are publicly
visible, so the confidentiality of encrypted note data depends on that protocol's
encryption scheme. Retrieval privacy is distinct: PIR can hide which records a
recipient requests from a retrieval service, but does not conceal the published
ciphertexts or, by itself, all network metadata.

A valid shielded proof does not establish that the intended recipient has
received or can recover the note opening. Privacy against network observers and
leakage through payment metadata also depend on the higher-level payment
protocol.[^tachyon-payment-protocols]

# Non-requirements

The Tachyon shielded protocol is unopinionated about the payment protocols
constructed on top of it. This ZIP does not prescribe payment-address formats,
note encryption, incoming-note discovery, selective disclosure, or a retrieval
mechanism.

On-chain publication and off-chain retrieval are separate concerns. A payment
protocol may publish encrypted notes on the ledger and retrieve them through
off-chain services, or use out-of-band note transmission instead. This ZIP
requires neither PIR nor a particular payment protocol; the resulting shielded
transactions still satisfy Tachyon's consensus rules.[^tachyon-payment-protocols]

# Deployment

The Tachyon shielded protocol will be deployed with
[NuTachyon](draft-tachyon-nutachyon-upgrade.md).


# Reference Implementation

The [Tachyon repository](https://github.com/tachyon-zcash/tachyon) implements the
shielded protocol in Rust, using the recursive proof-carrying data framework in
the [Ragu repository](https://github.com/tachyon-zcash/ragu). Experimental
full-node integration is provided by
[Zakura PR #795](https://github.com/zakura-core/zakura/pull/795), with corresponding
transaction-format, digest, value-accounting, and history-tree support in [zakura-core/common PR #178](https://github.com/zakura-core/common/pull/178).

# References

[^BCP14]: [Information on BCP 14 — RFC 2119 and RFC 8174](https://www.rfc-editor.org/info/bcp14)

[^protocol-tachyon]: Zcash Protocol Specification, Tachyon protocol. TODO: Add the version and section references once the Tachyon specification is written.

[^protocol-tachyon-epochs]: Zcash Protocol Specification, Tachyon epoch boundaries and cross-epoch validation rules. TODO: Add the version and section references once specified.

[^protocol-tachyon-proof-tree]: Zcash Protocol Specification, Tachyon PCD step statements and composition rules. TODO: Add the version and section references once specified.

[^protocol-tachyon-stamp-verification]: Zcash Protocol Specification, Tachyon stamp public inputs and proof verification. TODO: Add the version and section references once specified.

[^protocol-valuecommit]: [Zcash Protocol Specification: Homomorphic Pedersen commitments (Sapling and Orchard)](protocol/protocol.pdf#concretehomomorphiccommit)

[^protocol-tachyon-notecommit]: Zcash Protocol Specification, Tachyon note structure and commitment construction. TODO: Add the version and section references once specified.

[^protocol-tachyon-nullifiers]: Zcash Protocol Specification, Tachyon nullifier derivation and proof constraints. TODO: Add the version and section references once specified.

[^protocol-tachyon-consensus]: Zcash Protocol Specification, Tachyon transaction and block consensus rules. TODO: Add the version and section references once specified.

[^protocol-tachyon-multisetcommit]: Zcash Protocol Specification, Tachyon multiset commitment construction. TODO: Add the version and section references once specified.

[^protocol-tachyon-accumulator]: Zcash Protocol Specification, Tachygram accumulator and pool-state anchor updates. TODO: Add the version and section references once specified.

[^protocol-tachyon-keys]: Zcash Protocol Specification, Tachyon key components and derivation. TODO: Add the version and section references once specified.

[^tachyon-keys]: [The Tachyon Book: Key Hierarchy](https://github.com/tachyon-zcash/tachyon/blob/9abdcec1a98f96d91e612b3a43f5a4140de989f0/book/src/keys.md)

[^tachyon-secret-distribution]: Sean Bowe. [Tachyaction at a Distance](https://seanbowe.com/blog/tachyaction-at-a-distance/), May 15, 2025.

[^tachyon-payment-protocols]: [The Tachyon Book: A Deep Dive on Tachyon — Decoupling Payment Protocol from Shielded Protocol](https://github.com/tachyon-zcash/tachyon/blob/9abdcec1a98f96d91e612b3a43f5a4140de989f0/book/src/revisit.md)

[^tachyon-key-derivation]: [Tachyon reference implementation: Key derivation](https://github.com/tachyon-zcash/tachyon/blob/9abdcec1a98f96d91e612b3a43f5a4140de989f0/crates/tachyon/src/keys/private.rs)

[^tachyon-tachygrams]: [The Tachyon Book: Tachygrams](https://github.com/tachyon-zcash/tachyon/blob/9abdcec1a98f96d91e612b3a43f5a4140de989f0/book/src/tachygrams.md)

[^tachyon-nullifiers]: [The Tachyon Book: Nullifiers](https://github.com/tachyon-zcash/tachyon/blob/9abdcec1a98f96d91e612b3a43f5a4140de989f0/book/src/nullifiers.md)

[^evolving-nullifiers]: Sean Bowe and Ian Miers. [A Note on Notes: Towards Scalable Anonymous Payments via Evolving Nullifiers and Oblivious Synchronization](https://eprint.iacr.org/2025/2031). Cryptology ePrint Archive, Paper 2025/2031, 2025.

[^tachyon-notes]: [The Tachyon Book: Notes](https://github.com/tachyon-zcash/tachyon/blob/9abdcec1a98f96d91e612b3a43f5a4140de989f0/book/src/notes.md)

[^tachyon-note-implementation]: [Tachyon reference implementation: Notes and note commitments](https://github.com/tachyon-zcash/tachyon/blob/9abdcec1a98f96d91e612b3a43f5a4140de989f0/crates/tachyon/src/note.rs)

[^tachyon-authorization]: [The Tachyon Book: Authorization — Value Balance](https://github.com/tachyon-zcash/tachyon/blob/9abdcec1a98f96d91e612b3a43f5a4140de989f0/book/src/authorization.md#value-balance)

[^tachyon-multisetcommit]: [Tachyon reference implementation: Multiset commitments](https://github.com/tachyon-zcash/tachyon/blob/9abdcec1a98f96d91e612b3a43f5a4140de989f0/crates/tachyon/src/primitives/sets.rs)

[^tachyon-proof-tree]: [The Tachyon Book: Proof tree](https://github.com/tachyon-zcash/tachyon/blob/9abdcec1a98f96d91e612b3a43f5a4140de989f0/book/src/proof-tree.md)

[^tachyon-bundle-payload]: [The Tachyon Book: Bundle body and opaque payload](https://github.com/tachyon-zcash/tachyon/blob/9abdcec1a98f96d91e612b3a43f5a4140de989f0/book/src/bundle.md)

[^zip-0224]: [ZIP 224: Orchard Shielded Protocol](zip-0224.rst)

[^protocol-pallasandvesta]: [Zcash Protocol Specification: Pallas and Vesta](protocol/protocol.pdf#pallasandvesta)

[^zip-0225]: [ZIP 225: Version 5 Transaction Format](zip-0225.rst)

[^ragu]: [Ragu proving system](https://tachyon.z.cash/ragu/)

[^bootle16]: Jonathan Bootle, Andrea Cerulli, Pyrros Chaidos, Jens Groth, and Christophe Petit. [Efficient Zero-Knowledge Arguments for Arithmetic Circuits in the Discrete Log Setting](https://eprint.iacr.org/2016/263). Cryptology ePrint Archive, Paper 2016/263, 2016.

[^bclms21]: Benedikt Bünz, Alessandro Chiesa, William Lin, Pratyush Mishra, and Nicholas Spooner. [Proof-Carrying Data without Succinct Arguments](https://eprint.iacr.org/2020/1618). Cryptology ePrint Archive, Paper 2020/1618, 2021.
