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

Tachyon introduces [dynamic fees](#dynamicfees), allowing a prover to adjust fees
to network conditions within a wallet-authorized maximum without obtaining new
transaction signatures, including from an offline signing device.

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

## Tachygrams

Tachyon unifies the on-chain representation and consensus treatment of note
commitments and nullifiers, referring to both as _tachygrams_.

A tachygram is a Pallas base-field element representing a note commitment, a
nullifier, or a padding value. A spend contributes nullifiers for its epoch and
the next; an output contributes a note commitment and a padding tachygram to the stamp's tachygram multiset. Both
action types contribute exactly two values, so the tachygram count alone does not
reveal the split between spends and outputs. Each covered bundle also contributes
one [fee change note commitment](#dynamicfees).

Each action’s tachygrams are published as an untyped pair in its originating
bundle, with the bundle’s fee change note commitment in a separate field. The
zero-knowledge proof enforces their correct derivation and association with the
covered actions and fees.
Consensus therefore need not distinguish note commitments from nullifiers or
padding: it verifies the proof and applies the same duplicate-rejection
rules and accumulator updates to all tachygrams.[^tachyon-tachygrams]


## Stamp

A stamp packages a recursive zero-knowledge proof, a pool-state anchor, a
multiset commitment to the covered [tachygrams](#tachygrams), and the `txid`s of
other transactions whose bundles it covers. The anchor identifies the pool state
referenced by the proof. The proof binds the covered actions and fee descriptors
to the tachygrams and establishes the corresponding note-creation, spendability,
and fee-change statements.[^protocol-tachyon-stamp-verification]

A stamp can cover a single transaction or aggregate multiple transactions.
[Aggregation](#prooftree) replaces the input stamps with a single stamp, while
actions, signatures, tachygrams, and fee data remain in their respective bundle
bodies.[^tachyon-proof-tree] The bundle-format ZIP specifies the
[adjunct list](draft-tachyon-bundle-format.md#proofstamp) and how
[block validation](draft-tachyon-bundle-format.md#blockvalidity) checks complete
coverage.


## Epochs

An epoch is a fixed-length interval of consecutive blocks, identified by an index derived from block height. Each note has a deterministic nullifier for each epoch, independent of the choice of anchor. A spend referencing a pool state in epoch $e$ publishes the note’s nullifiers for epochs $e$ and $e+1$, allowing inclusion across an epoch boundary.[^tachyon-nullifiers]

When validating a transaction, validators check for duplicate nullifiers in the
current and immediately preceding epochs. Recursive spendability proofs cover
earlier history, allowing validators to prune older nullifier state from their
active duplicate-checking set.
[^tachyon-tachygrams] [^tachyon-proof-tree]

Epoch boundaries and cross-epoch validation rules MUST be implemented as specified
in the Zcash Protocol Specification.[^protocol-tachyon-epochs]


## Proof tree

Instead of Orchard's single action circuit, Tachyon uses a recursive proof tree of
specialized proof-carrying data (PCD) steps. Each step enforces its own constraints and recursively verifies up to two input PCD proofs, producing a new folded PCD proof.

Spend proofs combine evidence of note creation, correct per-epoch nullifier
derivation, and continued unspentness up to an anchor. Separate output steps prove
correct construction of new notes. The action proofs are combined at a common
anchor before applying the [dynamic fee mechanism](#dynamicfees) to produce a
stamp. The proof binds its anchor and multiset commitments to the covered action
and fee digests and tachygrams.

Stamps at a common anchor can be merged recursively to aggregate independently
constructed transactions. Before merging, anyone with the published transaction and chain data (likely a miner or other aggregator) can use anchor-chain proofs to lift stamps to a later anchor within the same spending epoch, enabling them to be merged during aggregation. Validators check the resulting proof against commitments reconstructed from the
covered bundle bodies, and check the anchor against chain state.

- PCD step statements and composition rules: [^protocol-tachyon-proof-tree]
- Stamp public inputs and verification: [^protocol-tachyon-stamp-verification]
- Design description: [^tachyon-proof-tree]

## Commitment schemes

Tachyon retains Orchard's homomorphic Pedersen value commitments on Pallas,
including the value and randomness generators. A Tachyon action commits to a
signed value, positive for spends and negative for outputs.

For the note structure and Poseidon note commitment, see
[Notes and note commitments](#notesandnotecommitments).

- Value commitment scheme: [^protocol-valuecommit]
- Design and implementation: [^tachyon-authorization]

### Multiset commitments

Tachyon uses deterministic Pedersen polynomial commitments on Vesta for
action-digest and tachygram multisets. A multiset of Pallas base-field elements is
encoded by taking its members as the roots of a monic polynomial. For action and
[fee](#dynamicfees) digests $d_i$ and tachygrams $t_j$:

$$A(X) = \prod_i \bigl(X - d_i\bigr) \qquad
T(X) = \prod_j \bigl(X - t_j\bigr)$$

The commitment to a polynomial of degree $n$ is the deterministic, untrapdoored
Pedersen commitment to its $n+1$ coefficients over the Vesta group, using the
polynomial-commitment generators fixed by Ragu, with no blinding term. The
coefficients are ordered by ascending degree: the constant term pairs with $G_0$,
and the degree-$k$ coefficient pairs with $G_k$.

A polynomial is invariant under permutation of its roots, and a repeated member
becomes a repeated root. These commitments therefore preserve multiplicity but
not order. They are deterministic functions of the public multisets and reveal
no information beyond those multisets.

The bundle-format ZIP defines [action digests](draft-tachyon-bundle-format.md#actiondigests)
that bind each action descriptor to its unordered tachygram pair, and
[fee digests](draft-tachyon-bundle-format.md#feedescriptorsanddigests) that bind each
fee descriptor to its fee change note commitment.

The multiset commitment construction MUST be implemented as specified in the
Zcash Protocol Specification.[^protocol-tachyon-multisetcommit]
Design and implementation: [^tachyon-multisetcommit].


## Tachygram accumulator

Tachyon unifies the history represented by Orchard's note commitment tree and
nullifier set in a single accumulator over [tachygrams](#tachygrams): note
commitments, nullifiers, and padding values.

The construction is a Poseidon hash chain of smaller multiset accumulators, one
per stamp. Each stamp's tachygram-multiset commitment is absorbed into the chain,
which also records epoch transitions, to derive pool-state anchors. Validators
still retain recent tachygrams for the duplicate checks described under
[Epochs](#epochs).

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
- The per-action randomizer $\alpha$ MUST be derived from the note commitment
  $\mathsf{cm}$ and per-action entropy using a domain-separated derivation. For a
  spend, the randomized spend-validating key
  $\mathsf{rk} = \mathsf{ak} + [\alpha]\mathcal{G}$ therefore binds the action's
  signed effecting data to the underlying note. Its tachygrams are carried as
  authorization data in the bundle body.[^tachyon-action-authorization]
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
the same action.[^tachyon-notes]

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
nullifiers were absent between its creation and that state; validators check for their non-existence in the retained tachygrams. This combines proofs of historical unspentness with a prunable consensus nullifier
set.[^tachyon-proof-tree]

Nullifier derivation and its proof constraints MUST be implemented as specified
in the Zcash Protocol Specification.[^protocol-tachyon-nullifiers]


## Dynamic fees

Each bundle authorizes a maximum fee through a signed value commitment to a
_max fee note_. The commitment binds the maximum amount and the payment key and
other fields used for fee change. The max fee note is never published as a
spendable note.

A prover chooses an actual fee between zero and the maximum, and creates a
_fee change note_ by subtracting that fee from the max fee note's value while
retaining its other fields. Its commitment and the actual fee are published in
the originating bundle's body and retained when its stamp is stripped. The
commitment is a tachygram even when the change value is zero. Changing the fee
requires a new stamp proof but no new transaction signatures.

The stamp covers a _fee descriptor_ comprising the maximum-fee value commitment
and actual fee for each covered bundle. Its proof enforces the fee range and
correct construction of the change note. The signed value balance counts the
maximum fee as an output; adding the actual fee gives the bundle's net pool
outflow. A bundle with no actions has zero maximum and actual fees.[^tachyon-fees]

The dynamic fee mechanism MUST be implemented as specified in the Zcash Protocol
Specification.[^protocol-tachyon-fees]


## Consensus rules

Tachyon consensus validation MUST be implemented as specified in the Zcash
Protocol Specification.[^protocol-tachyon-consensus] The following summary groups
the principal checks by what they validate: each transaction's bundle, or a stamp covering
one or more bundles. Aggregation combines stamp proofs, but does not eliminate
the checks on individual bundles.

### Per-bundle checks

The bundle-format ZIP's [Bundle validity](draft-tachyon-bundle-format.md#bundlevalidity)
rules apply to every bundle, whether proof-stamped or stripped. In summary:

- Bundle fields have canonical encodings and satisfy their type and range
  constraints. Each action's value commitment and authorization key are
  non-identity Pallas points, as is the maximum-fee value commitment. A bundle
  with no actions has zero value balance and zero maximum and actual fees.
- The tachygrams in the action pairs and fee change note commitment are distinct.
- Every action signature and the bundle's binding signature verify over the
  containing transaction's signature hash. The binding signature enforces
  consistency between the action value commitments, the maximum-fee value
  commitment, and the declared value balance.
- Opaque payment-protocol payloads satisfy the
  [canonical encoding rules](draft-tachyon-bundle-format.md#canonicalencodings), and
  their bytes are committed by the transaction's signature hash as described under
  [Transaction digest contributions](draft-tachyon-bundle-format.md#transactiondigestcontributions).

### Per-stamp checks

These checks apply to each proof-bearing stamp, using all bundles it covers.
The bundle-format ZIP's [Block validity](draft-tachyon-bundle-format.md#blockvalidity)
section specifies block-scoped coverage and proof-verification procedures.
The anchor, epoch-window, and accumulator rules below additionally depend on chain
state.

- Each proof stamp covers its own bundle and the stripped bundles identified by
  its adjunct txid list in the same block. Every stripped bundle is listed by
  exactly one proof stamp. The covered action descriptors and fee descriptors
  are each distinct.
- Each stamp's tachygram multiset is reconstructed from the action pairs and fee
  change note commitments in its covered bundle bodies. Its carried commitment
  matches this multiset, and its proof verifies against the anchor and
  reconstructed multiset commitments.
- Each stamp anchor identifies an accepted end-of-block pool state in the
  epoch of the block being validated or the immediately preceding epoch.
- Tachygrams are distinct across all bundle bodies in the block and from
  tachygrams published in earlier blocks of the current or immediately preceding
  epoch. This check treats note commitments, nullifiers, and padding identically.
- Each stamp advances the pool-state accumulator in transaction order, with
  epoch transitions incorporated as specified under
  [Tachygram accumulator](#tachygramaccumulator).

# Privacy and Security Implications

Each action's tachygram pair and each bundle's fee change note commitment are
publicly associated with their originating action or bundle. Comparing versions
of the same transaction re-proved for different spending epochs distinguishes
spends, whose pairs change, from outputs, whose pairs remain fixed. A fee
adjustment alone leaves action pairs unchanged.[^tachyon-tachygrams] A delegate
that knows a note's nullifier window can recognize its spending action from the
published pair.[^tachyon-nullifiers]

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

This ZIP does not prescribe P2P aggregation messages, covered-transaction discovery,
or mempool and relay policy. 

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

[^protocol-tachyon-fees]: Zcash Protocol Specification, Tachyon dynamic fees, fee change notes, and proof constraints. TODO: Add the version and section references once specified.

[^protocol-tachyon-consensus]: Zcash Protocol Specification, Tachyon transaction and block consensus rules. TODO: Add the version and section references once specified.

[^protocol-tachyon-multisetcommit]: Zcash Protocol Specification, Tachyon multiset commitment construction. TODO: Add the version and section references once specified.

[^protocol-tachyon-accumulator]: Zcash Protocol Specification, Tachygram accumulator and pool-state anchor updates. TODO: Add the version and section references once specified.

[^protocol-tachyon-keys]: Zcash Protocol Specification, Tachyon key components and derivation. TODO: Add the version and section references once specified.

[^tachyon-keys]: [The Tachyon Book: Key Hierarchy](https://github.com/tachyon-zcash/tachyon/blob/9abdcec1a98f96d91e612b3a43f5a4140de989f0/book/src/keys.md)

[^tachyon-secret-distribution]: Sean Bowe. [Tachyaction at a Distance](https://seanbowe.com/blog/tachyaction-at-a-distance/), May 15, 2025.

[^tachyon-payment-protocols]: [The Tachyon Book: A Deep Dive on Tachyon — Decoupling Payment Protocol from Shielded Protocol](https://github.com/tachyon-zcash/tachyon/blob/9abdcec1a98f96d91e612b3a43f5a4140de989f0/book/src/revisit.md)

[^tachyon-key-derivation]: [Tachyon reference implementation: Key derivation](https://github.com/tachyon-zcash/tachyon/blob/9abdcec1a98f96d91e612b3a43f5a4140de989f0/crates/tachyon/src/keys/private.rs)

[^tachyon-tachygrams]: [The Tachyon Book: Tachygrams](https://github.com/tachyon-zcash/tachyon/blob/9d9af51a9c4e8ba1e19f5485eafe066e46137be9/book/src/tachygrams.md)

[^tachyon-nullifiers]: [The Tachyon Book: Nullifiers](https://github.com/tachyon-zcash/tachyon/blob/9d9af51a9c4e8ba1e19f5485eafe066e46137be9/book/src/nullifiers.md)

[^evolving-nullifiers]: Sean Bowe and Ian Miers. [A Note on Notes: Towards Scalable Anonymous Payments via Evolving Nullifiers and Oblivious Synchronization](https://eprint.iacr.org/2025/2031). Cryptology ePrint Archive, Paper 2025/2031, 2025.

[^tachyon-notes]: [The Tachyon Book: Notes](https://github.com/tachyon-zcash/tachyon/blob/9abdcec1a98f96d91e612b3a43f5a4140de989f0/book/src/notes.md)

[^tachyon-note-implementation]: [Tachyon reference implementation: Notes and note commitments](https://github.com/tachyon-zcash/tachyon/blob/9abdcec1a98f96d91e612b3a43f5a4140de989f0/crates/tachyon/src/note.rs)

[^tachyon-fees]: [The Tachyon Book: Dynamic Fee](https://github.com/tachyon-zcash/tachyon/blob/9d9af51a9c4e8ba1e19f5485eafe066e46137be9/book/src/fee.md)

[^tachyon-authorization]: [The Tachyon Book: Authorization — Value Balance](https://github.com/tachyon-zcash/tachyon/blob/9abdcec1a98f96d91e612b3a43f5a4140de989f0/book/src/authorization.md#value-balance)

[^tachyon-action-authorization]: [The Tachyon Book: Authorization — Per-action Signing](https://github.com/tachyon-zcash/tachyon/blob/9abdcec1a98f96d91e612b3a43f5a4140de989f0/book/src/authorization.md#per-action-signing)

[^tachyon-multisetcommit]: [Tachyon reference implementation: Multiset commitments](https://github.com/tachyon-zcash/tachyon/blob/9abdcec1a98f96d91e612b3a43f5a4140de989f0/crates/tachyon/src/primitives/sets.rs)

[^tachyon-proof-tree]: [The Tachyon Book: Proof tree](https://github.com/tachyon-zcash/tachyon/blob/9d9af51a9c4e8ba1e19f5485eafe066e46137be9/book/src/proof-tree.md)

[^zip-0224]: [ZIP 224: Orchard Shielded Protocol](zip-0224.rst)

[^protocol-pallasandvesta]: [Zcash Protocol Specification: Pallas and Vesta](protocol/protocol.pdf#pallasandvesta)

[^zip-0225]: [ZIP 225: Version 5 Transaction Format](zip-0225.rst)

[^ragu]: [Ragu proving system](https://tachyon.z.cash/ragu/)

[^bootle16]: Jonathan Bootle, Andrea Cerulli, Pyrros Chaidos, Jens Groth, and Christophe Petit. [Efficient Zero-Knowledge Arguments for Arithmetic Circuits in the Discrete Log Setting](https://eprint.iacr.org/2016/263). Cryptology ePrint Archive, Paper 2016/263, 2016.

[^bclms21]: Benedikt Bünz, Alessandro Chiesa, William Lin, Pratyush Mishra, and Nicholas Spooner. [Proof-Carrying Data without Succinct Arguments](https://eprint.iacr.org/2020/1618). Cryptology ePrint Archive, Paper 2020/1618, 2021.
