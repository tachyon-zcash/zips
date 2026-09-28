```
ZIP: XXX
Title: Tachyon Aggregator Protocol
Owners: Tachyon Team (tachyon.z.cash)
Status: Draft
Category: Network
Created: 2026-06-26
License: MIT
Discussions-To: <https://github.com/tachyon-zcash/tachyon/issues/106>
```

# Terminology

The key words "MUST", "MUST NOT", "SHOULD", "SHOULD NOT", "MAY", and "RECOMMENDED" in this document are to be interpreted as described in BCP 14 [^BCP14] when, and only when, they appear in all capitals.

The term "network upgrade" is to be interpreted as described in ZIP 200.[^zip-0200]
The terms "Testnet" and "Mainnet" are to be interpreted as described in § 3.12 ‘Mainnet and Testnet’.[^protocol]

The terms "tachygram" and "stamp" are defined by the [Tachyon Shielded Protocol](draft-tachyon-shielded-protocol.md) and [Tachyon Bundle / Aggregate Transaction Format](draft-tachyon-bundle-format.md) ZIPs respectively, and are summarized here non-normatively.
The remaining terms are defined by this ZIP.

Tachygram
:   The `byte[32]` encoding of a field element ($\mathbb{F}_p$) representing either a note nullifier or a note commitment.
    Consensus treats nullifiers and commitments identically.

Stamp
:   The final section of a bundle.
    Either a proof stamp, carrying a Ragu proof and some supporting data, or a pointer stamp, carrying a `wtxid` reference to a transaction with a proof.

Aggregator
:   A participant that merges two stamps into one covering stamp and publishes the result.
    The role is permissionless and carries no protocol-level exclusivity.

*Autonome*
:   A stand-alone transaction whose bundle bears an independent proof stamp covering only its own actions (an *autonome* bundle).
    The standard form of a user-originated Tachyon transaction.
    An *autonome* may appear in a block or in the mempool.

*Aggregate*
:   A transaction whose bundle bears a merged proof stamp covering other transactions (an *aggregate* bundle).
    An *aggregate* MAY contain zero or more Tachyon actions of its own.
    An *aggregate* may appear in a block or in the mempool.

*Adjunct*
:   An abbreviated transaction that only appears within a block.
    An *adjunct*'s bundle has been stripped of its original stamp and now bears a pointer stamp.
    *Adjuncts* retain action data, action signatures, binding signature, `valueBalanceTachyon`, and `vMemoTachyon`.

# Abstract

Tachyon shielded transactions use a recursive proof system.
Recursion allows many per-transaction proofs to be combined into one covering proof, reducing the number of separate proofs carried on-chain and verified during validation and syncing.
This recursion admits a new participant role, the aggregator, without creating a new trust assumption.

This ZIP specifies the aggregator protocol: an 8-step lifecycle from transaction authorization, through the mempool, to block layout and final validation.
It comprises a block-layout discipline under which miners replace a covered bundle's proof with a reference to a covering transaction, the effecting-data and authorizing-data semantics that make stripping safe, and P2P rules extending ZIP 239 to Tachyon's authorization-form malleability.
Aggregation changes the stamp, not the bundle body.
Effecting data (action descriptors, value balance, and memo payload) and the action and binding signatures remain present when a proof stamp is replaced by a pointer stamp.
Signature and balance verification remain per-bundle.

# Motivation

Consensus requires every bundle to be verified.
Without aggregation, each Tachyon bundle carries a separate proof.
Aggregation amortizes proof data and verification across covered transactions.

The proof system permits public aggregation of already-published proofs, so the aggregator is a permissionless, conceptual role that any participant may take, not a designated prover.
Aggregation reduces the number of stamp proofs a validator verifies, but the public-data, signature, and balance checks still apply.

Nodes need not hold identical mempools. A receiver may therefore need to obtain
missing covered transactions before it can validate, relay, or further aggregate
an advertised *aggregate*. The advertising peer provides this dependency data;
recovery does not require proof that the transactions were previously broadcast.

Aggregation is optional.
Miners remain free to include non-aggregated Tachyon transactions; any *aggregate* a block does contain must be fully backed by *adjuncts* in the same block.

# Requirements

* Reduce per-block stamp-verification cost by allowing multiple transactions' stamps to be merged into one *aggregate* stamp.
* Confine aggregation to the stamp: retain each bundle's action descriptors, value balance, memo payload, and signatures, with signature and balance verification unchanged and per-bundle.
* Preserve transaction-identifier stability: a transaction's `txid` is invariant across stamping, merging, and stripping.
* Allow any participant to act as aggregator; no protocol-level exclusivity.
* Enable a validator or miner to confirm they hold all necessary data before attempting proof verification.
* Enable recovery of missing covered transactions while bounding dependency-serving
  and retrieval work.
* Introduce no new trust assumption: every invariant is enforced either by proof or by consensus rules.

# Specification

The specification is organized around the 8-step aggregation lifecycle.
Each step is a subsection with conformance language.
Two cross-cutting concerns are specified in their own subsections after the lifecycle and referenced from the steps that invoke them: [Transaction identifiers and P2P relay](#transactionidentifiersandp2prelay), and [Covered-transaction identification](#covered-transactionidentification).

## Step 1: Publication of autonomes

Transaction authors produce complete and independently verifiable *autonome* transactions with proof stamps and broadcast them to the mempool via the existing wallet RPC or P2P path.

## Step 2: Aggregator observation and selection

Aggregators observe transactions in mempool gossip (see [Transaction identifiers and P2P relay](#transactionidentifiersandp2prelay)).

An aggregator selects two transactions (*autonomes* or existing *aggregates*) for merging into a new *aggregate*.
Selected transactions MUST bear anchors that are identical or alignable by lifting (see [Step 3](#step3:witnessandpcdpreparation)).
Selected transactions SHOULD bear disjoint tachygram sets; overlapping selections cannot yield a valid *aggregate*, so this guidance needs no enforcement (see [Overlapping merges self-invalidate](#overlappingmergesself-invalidate)).

## Step 3: Witness and PCD preparation

The aggregator reconstructs each selected stamp PCD from its proof, anchor, tachygram commitment, and a reconstructed commitment to its covered actions.
If both selected transactions are *autonomes*, their covered action data is directly available on the transactions themselves.

Merging is defined only over stamps bearing identical anchors.
If the selected transactions bear unequal anchors, the aggregator first aligns them by lifting the older anchor to the newer with a lift PCD that proves the anchor sequence between them.

Lifting is confined to the anchor's epoch.
A stamp's proof fixes each spend's published nullifiers relative to the epoch of its anchor (see the [Tachyon statement](https://github.com/turbocrime/tachyon/blob/2840ed7dc7b0dee5401b267d04a8c3da3ee026c3/book/src/zips/tachyon-shielded-protocol.md#tachyon-statement)), so a lift never crosses an epoch boundary, and stamps bearing anchors of different epochs cannot be aligned for merging.

A selected transaction that is already an *aggregate* additionally requires the actions of every transaction contributing to it.
The *aggregate* does not carry its contributors' transaction identifiers.
An aggregator uses [covered-transaction identification](#covered-transactionidentification)
over locally held transactions and requests missing dependency data from the
advertising peer as described in
[Aggregate dependency availability](#aggregatedependencyavailability).

Aggregators maintain an index of recent transactions, along with recent consensus
data and cached anchor lift proofs, and do not attempt a merge until the required
covered transaction data is available and validated.

## Step 4: Aggregate construction

Holding two stamp PCD with identical anchors and the prepared witness, the aggregator executes a merge, proving a new stamp PCD that covers the selected transactions and every transaction contributing to them.

The aggregator MAY carry the merged stamp on a newly constructed transaction, or update either contributing transaction in place, replacing its stamp with the merged stamp.

## Step 5: Aggregate publication

The aggregator publishes the *aggregate* transaction to the mempool.
A newly constructed transaction has a new `txid` and `wtxid`.
Replacing a transaction's stamp preserves its `txid` and changes its `wtxid`.
Relay follows [Transaction identifiers and P2P relay](#transactionidentifiersandp2prelay).
Advertising the *aggregate* also incurs the
[dependency-serving obligation](#aggregatedependencyavailability).

## Step 6: Miner observation and selection

Miners observe both *autonomes* and *aggregates* in the mempool (see [Transaction identifiers and P2P relay](#transactionidentifiersandp2prelay)) and select the *aggregates* and the transactions they cover to include in a block.
Determining which transactions a candidate *aggregate* covers is [covered-transaction identification](#covered-transactionidentification).
A miner MAY also vertically integrate aggregation, producing its own *aggregates* privately during block assembly rather than sourcing them from the mempool.

## Step 7: Block assembly

Use of *aggregates* within a block is RECOMMENDED, not required.

Miners MAY perform additional aggregation during block assembly, without publishing the *aggregate* to the mempool.
The resulting *aggregate* is included directly in the miner's proposed block.
The merge and anchor-alignment rules of Steps 3 and 4 apply unchanged.

A block MAY contain, in any combination:

* zero or more non-Tachyon transactions
* zero or more Tachyon *autonomes*
* zero or more Tachyon *aggregates*

A block containing an *aggregate* MUST also contain, as *adjuncts*, every transaction that *aggregate* covers.
For each such *adjunct*, the miner MUST strip the bundle's stamp and set the `tachyonAggregateId` field to the covering *aggregate*'s `wtxid`.

An *adjunct* bundle with no actions of its own still names a covering transaction.
Its `tachyonAggregateId` MUST identify a proof-stamped transaction in the same block, and SHOULD refer to the *aggregate* that ultimately absorbed its stamp.

## Step 8: Block validation

Block validation closes the aggregation lifecycle: a validator confirms that every proof stamp in a block is backed by the transactions it covers and that every proof holds.
The normative consensus rules for this are specified by [Block validity](draft-tachyon-bundle-format.md#blockvalidity) in the Tachyon Bundle / Aggregate Transaction Format ZIP; this step describes only how those rules fit the lifecycle.

Validation covers tachygram distinctness across the block, association of each *adjunct* with the *aggregate* covering it, confirmation of each stamp's covered-actions digest against the actions actually present in the block, and verification of every proof.
Reuse of a tachygram across the wider epoch window is a separate consensus concern owned by the [Tachyon Accumulator / Hash Chain ZIP](https://github.com/turbocrime/tachyon/blob/2840ed7dc7b0dee5401b267d04a8c3da3ee026c3/book/src/zips/tachyon-accumulator.md#epoch-window).

The spirit of these checks is fail-fast ordering: the cheap public-data scans (tachygram distinctness, *adjunct* association, and covered-actions confirmation) run before the costly proof verification, so a block that violates a cheaper rule is rejected without any proof being verified.

## Transaction identifiers and P2P relay

These semantics underpin publication (Step 5), observation (Steps 2 and 6), and stripping (Step 7).

### Identifiers

A Tachyon transaction is identified by `wtxid = txid || auth_digest` (ZIP 239 [^zip-0239], ZIP 244 [^zip-0244]).
The digest inputs (which fields each contribution commits to) are defined normatively by the [Tachyon Bundle / Aggregate Transaction Format](draft-tachyon-bundle-format.md) ZIP, and the digest algorithm by the [ZIP 244 update](https://github.com/turbocrime/tachyon/blob/2840ed7dc7b0dee5401b267d04a8c3da3ee026c3/book/src/zips/zip-244.md); this section summarizes them non-normatively.

* Tachyon's contribution to `txid` commits to `hActionsTachyon || valueBalanceTachyon || hMemoTachyon`, and excludes the stamp.
It is stable across stamping, merging, stripping, and re-stamping, so a transaction's logical identity is invariant across the aggregation lifecycle.
* `auth_digest` commits to action signatures, the binding signature, and the stamp.
A proof stamp contributes a `byte[64]` stamp digest; a pointer stamp contributes the `byte[64]` covering `wtxid`.
The `"ZTxAuthTachyHash"` personalization and the normative digest algorithm are specified by the [ZIP 244 update](https://github.com/turbocrime/tachyon/blob/2840ed7dc7b0dee5401b267d04a8c3da3ee026c3/book/src/zips/zip-244.md).
* A transaction's effecting data fixes its `txid`, but the transaction can be authorized in multiple forms that share that `txid` and differ only in `auth_digest`, hence in `wtxid`: a wallet's *autonome*, an anchor-lifted or proof-rerandomized restamp, and the *adjunct* a miner produces.
The covering-*aggregate* reference an *adjunct* carries is a `wtxid`, not a `txid`, to pin a specific *aggregate*.
* The wire byte `tachyonBundleState` (`uint8`) distinguishes forms: `0x00` no Tachyon bundle, `0x01` proof stamp, `0x02` pointer stamp.

### Relay

Tachyon bundles are announced and fetched by `wtxid` using the `MSG_WTX` inv type, and nodes MUST treat distinct `wtxid`s as distinct inventory objects.
`MSG_WTX` relay is mandatory: restamping changes a transaction's `wtxid` while leaving `txid` unchanged, so announcement by `txid` alone could not distinguish the proof-stamped forms a node may be offered.

### Aggregate dependency availability

A node advertising an *aggregate* MUST have verified it and MUST retain and serve
the covered transaction data required to validate it for the dependency-serving
period. This obligation applies to every relaying node, not only the aggregator
that constructed it.

Dependency recovery is scoped to the advertised *aggregate*'s exact `wtxid`.
The advertising peer supplies a description of the covered transactions, allowing
the receiver to reuse locally held data and request only missing dependencies.
Dependency descriptions and responses are untrusted: the receiver MUST confirm
complete action coverage against `hStampActionsTachyon`, verify the stamp proof,
and perform signature, balance, and other applicable checks on the covered
transactions before admitting or relaying the *aggregate*.

Pending aggregates MUST remain outside the validated mempool and ordinary
transaction relay until verification completes. Dependency requests, responses,
and pending work MUST be bounded per peer and globally by count, bytes, time,
rate, and concurrency. A peer MAY refuse requests exceeding the serving limits;
such refusals MUST NOT be penalized as failures to honor the serving obligation.

Failure to obtain dependencies does not establish that an *aggregate* is invalid.
Repeated failures to serve valid, in-limit requests during the serving period MAY
reduce the advertising peer's availability score or service priority, or lead to
disconnection. Availability failures MUST be distinguished from invalid-proof or
invalid-transaction failures; an isolated timeout MUST NOT incur such a validation
penalty. Nodes MUST track the suppliers of aggregates and dependency responses
separately, so invalid data from another responder is not attributed to the
aggregate's sender. Scoring weights remain implementation policy.

TODO: Specify the dependency description and request/response encodings, serving
period and renewal rules, request limits, timeouts and retries, and the effects
of transaction expiry, reorgs, and advertisement withdrawal on serving obligations.
This subsection is not yet a complete wire protocol. Any stamp-stripped dependency
encoding needs separate reconstruction semantics; it must not be treated as
ordinary adjunct relay or as the original transaction's `MSG_WTX` object.

### Duplicate tachygrams are transaction-invalid

Tachygram distinctness applies at the transaction level: a proof stamp whose `vTachygrams` contains a duplicate tachygram is invalid, and a node MUST NOT accept it into the mempool or relay it.
The check is a scan of the public list, requiring no proof verification.

### Proof verification precedes relay

A node MUST verify a bundle's proof stamp before accepting the transaction into its mempool or relaying it.
This paragraph specifies only the Tachyon-proof-specific requirement; mempool acceptance also requires the standard checks (action and binding signature verification, balance rules, and general transaction validity).
The proof check needs the covered actions.
The node confirms `cTachygrams` against `vTachygrams`, then assembles the proof header from that commitment, `anchorTachyon`, and `cStampActionsTachyon` reconstructed from the covered actions' digests.
It verifies the proof against that header.
An *autonome* is self-contained, since its stamp covers only its own actions.
For an *aggregate* covering other transactions, the node collects the covered
actions from locally held transactions and dependency responses, and
checks that set against the carried `hStampActionsTachyon` (see
[Covered-transaction identification](#covered-transactionidentification)) before
assembling the header.
Without the covered actions, the *aggregate* cannot be verified.

### Adjunct bundles are forbidden from the mempool

A bundle in the *adjunct* state (`tachyonBundleState == 0x02`) MUST NOT be accepted into the mempool or published through transaction relay; it is valid only inside a block.
This restriction does not prohibit transmitting adjuncts as part of a block.
Transaction relay carries proof-stamped bundles (*autonomes* and *aggregates*) only.
An aggregator publishes its merged stamp as a new proof-stamped *aggregate*.
This relay policy and `MSG_WTX` operate at different layers: `MSG_WTX` fixes the identifier and inventory semantics, while the policy admits only the proof-stamped forms.

## Covered-transaction identification

Identifying which transactions a stamp covers is a single primitive, used by aggregators preparing a merge witness (Step 3) and by miners observing and composing a block (Steps 6 and 7).

Tachygrams give a first pass: an *aggregate*'s stamp publishes the tachygrams of every action it covers, so transactions whose tachygrams appear there are candidates.
Tachygram overlap only identifies possible candidates; it cannot confirm the collection is complete.

The *aggregate*'s stamp field `hStampActionsTachyon` is a coverage digest representing every action the *aggregate* covers.
A collection that reproduces `hStampActionsTachyon` indicates the verifier is prepared to execute proof verification.

# Rationale

This subsection is non-normative.

## Lifecycle-structured specification

Organizing the Specification around the lifecycle mirrors how participants actually move through the protocol.
The two concerns that several steps share, transaction-identifier and relay semantics and covered-transaction identification, are factored into their own subsections and referenced from the steps, so no step restates another's rules and the lifecycle reads as a sequence of actions.

## Cheap coverage confirmation

`hStampActionsTachyon` gives the fail-fast completeness check of [Covered-transaction identification](#covered-transactionidentification) at the cost of a sort and one hash.

## Overlapping merges self-invalidate

The disjoint-selection guidance of Step 2 is a SHOULD rather than a consensus rule because a violation cannot survive.
The merge proves the merged multiset polynomial to be the product of its two inputs', so an overlapping tachygram appears as a repeated root and the merged stamp commits to it twice.
The wire format admits only distinct tachygrams in canonical order, so no publishable `vTachygrams` reconstructs that commitment, and the *aggregate* has no serialization that both parses and verifies against its own proof (see [Block validity](draft-tachyon-bundle-format.md#blockvalidity)).
The aggregator's proving work is simply wasted, so the selection rule needs no enforcement of its own.

## Verified relay

Requiring proof verification before relay is standard mempool hygiene, and aggregation sharpens it: an invalid stamp is a dead end, since a merge cannot produce a valid *aggregate* from an invalid input and no valid block can carry an unverifiable proof (see [Block validity](draft-tachyon-bundle-format.md#blockvalidity)).
Aggregators would therefore hit every invalid proof themselves at the merge; verifying at relay pushes that discovery to the network edge rather than spending bandwidth propagating transactions no aggregator can use and no block can include.

## Stripping is miner-side, not relay-time

Stripping at relay time would force every relay node to understand *aggregate* coverage and would mix block-assembly policy with gossip.
Keeping the P2P network carrying proof-stamped bundles only, and forbidding *adjunct* bundles from the mempool, simplifies relay: an *adjunct* bundle has no valid wire form until its covering `wtxid` is assigned, which happens at block assembly.

## `MSG_WTX` relay is mandatory

Tachyon's authorization-form malleability creates the same need to distinguish witnesses that motivated ZIP 239.
`MSG_WTX` distinguishes authorization forms sharing a `txid`; it does not eliminate that malleability.

# Security Implications

This subsection is non-normative.

## No new trust assumption

The aggregator is not trusted.
Validators check the proof and the public-data consensus rules rather than relying on the aggregator's claims.
A malicious aggregator may publish an invalid *aggregate*, but block validation (see [Block validity](draft-tachyon-bundle-format.md#blockvalidity)) rejects it.
A miner's proposed block fails validation if any pointer stamp does not identify a proof-stamped transaction in the block, or if the collected actions fail the coverage or proof checks.
For an actionless adjunct, those checks do not establish which aggregate absorbed its former stamp.

## Data availability

Aggregation replaces covered proof stamps with pointers and publishes their combined tachygrams in the covering stamp.
Every *adjunct* retains its action data, action signatures, binding signature, `valueBalanceTachyon`, and `vMemoTachyon`; validators reconstruct the *aggregate* header from the action data.
An *aggregate* proof alone is insufficient: the covered effecting data is present in the block as *adjuncts*, and [Block validity](draft-tachyon-bundle-format.md#blockvalidity) rejects any block where it is not.

## Circuit/consensus boundary

Several security properties are enforced by consensus rather than fully proven inside the stamp.
Double-spend prevention rests on the block-scoped tachygram-uniqueness check (see [Block validity](draft-tachyon-bundle-format.md#blockvalidity)) together with the [epoch-window duplicate-tachygram](https://github.com/turbocrime/tachyon/blob/2840ed7dc7b0dee5401b267d04a8c3da3ee026c3/book/src/zips/tachyon-accumulator.md#epoch-window) and [anchor-membership](https://github.com/turbocrime/tachyon/blob/2840ed7dc7b0dee5401b267d04a8c3da3ee026c3/book/src/zips/tachyon-accumulator.md#anchor-membership) rules owned by the [Tachyon Accumulator / Hash Chain](https://github.com/turbocrime/tachyon/blob/2840ed7dc7b0dee5401b267d04a8c3da3ee026c3/book/src/zips/tachyon-accumulator.md) ZIP and the spendable-lineage rules owned by the [Tachyon Shielded Protocol](draft-tachyon-shielded-protocol.md) ZIP.
Those base-protocol rules are depended upon, not re-specified here, so implementers and auditors should not assume the proof alone establishes these properties.

# Privacy Implications

This subsection is non-normative.

## Privacy of aggregation relationships

An observer holding candidate transactions can use `vTachygrams` overlap and `hStampActionsTachyon` to infer an aggregate's coverage (see [Covered-transaction identification](#covered-transactionidentification)).
The aggregate alone does not carry all the covered action data needed to reconstruct its proof header.
Selective dependency requests can reveal which covered transactions a receiver lacks,
and responses disclose the serving peer's proposed transaction coverage. They do
not establish when or whether those transactions were previously broadcast.
The privacy of note contents depends on the Tachyon Shielded Protocol, not on the coverage check.
An *adjunct* bundle with no actions is valid against any claimed covering stamp (see [Block validity](draft-tachyon-bundle-format.md#blockvalidity)), so an observer reconstructing aggregation relationships cannot rely on an actionless bundle's reference.

# Deployment

This ZIP is deployed with a Tachyon network upgrade.
Activation parameters are specified by the corresponding deployment ZIP ([Network Upgrade Deployment](draft-tachyon-nutachyon-upgrade.md)).

# Reference implementation

The `zcash_tachyon` crate implements the bundle state machine, stamp merging, stripping, and the `hStampActionsTachyon` coverage check: <https://github.com/tachyon-zcash/tachyon>.

# References

[^BCP14]: [Information on BCP 14: "RFC 2119: Key words for use in RFCs to Indicate Requirement Levels" and "RFC 8174: Ambiguity of Uppercase vs Lowercase in RFC 2119 Key Words"](https://www.rfc-editor.org/info/bcp14)

[^protocol]: [Zcash Protocol Specification](protocol/protocol.pdf)

[^zip-0200]: [ZIP 200: Network Upgrade Mechanism](zip-0200.rst)

[^zip-0239]: [ZIP 239: Relay of Version 5 Transactions](zip-0239.rst)

[^zip-0244]: [ZIP 244: Transaction Identifier Non-Malleability](zip-0244.rst)
