```
ZIP: XXX
Title: Tachyon Bundle / Aggregate Transaction Format
Owners: Tachyon Team (tachyon.z.cash)
Status: Draft
Category: Consensus
Created: 2026-07-02
License: MIT
Discussions-To: <https://github.com/tachyon-zcash/tachyon/issues/104>
```

# Terminology

The key words "MUST", "MUST NOT", "SHOULD", "SHOULD NOT", "MAY", and "RECOMMENDED" in this document are to be interpreted as described in BCP 14 [^BCP14] when, and only when, they appear in all capitals.

The term "network upgrade" is to be interpreted as described in ZIP 200.[^zip-0200]
The terms "Testnet" and "Mainnet" are to be interpreted as described in § 3.12 ‘Mainnet and Testnet’.
The character § is used when referring to sections of the Zcash Protocol Specification.[^protocol]

`txid`, `auth_digest`, and the SIGHASH transaction hash are defined by ZIP 244 [^zip-0244]; `wtxid = txid || auth_digest`, the 64-byte identifier used for transaction announcement and relay, by ZIP 239 [^zip-0239].
Value commitments, spend authorization signatures, and binding signatures are existing constructions (§ 5.4.8.3 ‘Homomorphic Pedersen commitments (Sapling and Orchard)’, § 4.15 ‘Spend Authorization Signature (Sapling and Orchard)’, and § 4.14 ‘Balance and Binding Signature (Orchard)’, respectively, in their Orchard instantiations); this ZIP applies them to Tachyon as specified below rather than redefining them.

The following terms are defined by the
[Tachyon Shielded Protocol](draft-tachyon-shielded-protocol.md) ZIP and summarized here:

Tachygram
:   A Pallas base-field element representing a note commitment, a nullifier, or a
    padding value, as defined under
    [Tachygrams](draft-tachyon-shielded-protocol.md#tachygrams).
    Its `byte[32]` wire representation is specified under
    [Canonical encodings](#canonicalencodings).

Anchor
:   A Poseidon hash-chain state referencing the *Tachyon pool* at a specific block.
    The chain advances at sub-block granularity, but consensus acknowledges only
    end-of-block states as anchors, as described in the Tachyon Shielded Protocol ZIP
    under [Tachygram accumulator](draft-tachyon-shielded-protocol.md#tachygramaccumulator)
    and [Per-stamp checks](draft-tachyon-shielded-protocol.md#per-stampchecks).

Multiset commitment
:   As defined in [Multiset commitments](draft-tachyon-shielded-protocol.md#multisetcommitments)
    in the Tachyon Shielded Protocol ZIP. Its application to this format is
    described under [Multiset commitments](#multisetcommitments) below.

Stamp
:   The proof-bearing protocol object defined under
    [Stamp](draft-tachyon-shielded-protocol.md#stamp) in the Tachyon Shielded Protocol ZIP.
    This ZIP defines two wire forms for a bundle's final section: a proof stamp
    carrying that object, or an empty stripped stamp.

The remaining terms are defined by this ZIP:

Bundle
:   The Tachyon section of a transaction: a body carrying actions, signatures,
    tachygrams, value balance, and fee data, followed by a state-dependent stamp.

Bundle state
:   The three-valued wire discriminator `tachyonBundleState` selecting no bundle, a proof stamp, or a stripped stamp.

Action
:   The triple $(\mathsf{cv}, \mathsf{rk}, \mathsf{sig})$: a value commitment, a randomized verification key, and a signature over the transaction sighash.
    An action effects a spend or an output; both forms share this encoding, and consensus applies the same rules to each.

Action digest
:   The Poseidon digest binding an action's $(\mathsf{cv}, \mathsf{rk})$
    descriptor to its unordered tachygram pair.

Descriptor digest
:   The BLAKE2b-256 digest of a sequence of action descriptors (see [Action descriptor digests](#actiondescriptordigests)).

Proof stamp
:   A wire representation of a stamp, carrying a Ragu proof, an anchor, a
    tachygram multiset commitment, and the `txid`s of the other transactions it
    covers. The proof attests that every covered action and fee satisfies the
    Tachyon protocol rules.

Stripped stamp
:   An empty wire form used when another transaction carries the covering proof.

Covering transaction
:   The proof-stamped transaction whose stamp covers its own bundle and the
    stripped bundles identified by its adjunct list.

Adjunct
:   A transaction whose Tachyon bundle is covered by another transaction's stamp.
    Its bundle is stripped when included in a block.

# Abstract

This ZIP is a living draft and does not prescribe whether the Tachyon bundle is
deployed through ZIP 248's extensible transaction format[^zip-0248] or incorporated
directly into a new transaction format. Its encoding and integration details may
be revised as ZIP 248 and other related draft ZIPs evolve.

This ZIP specifies the consensus wire format of the Tachyon bundle: a three-state
discriminator byte, a body carrying actions, signatures, tachygrams, value balance,
and fee data, and either a proof stamp or an empty stripped stamp.
It defines the canonical field encodings and sequence orders, the action and fee
digests, and the use of the shielded protocol's multiset commitments in bundle verification.
It further defines the bundle's transaction-digest inputs, the consensus rules scoped to a single bundle, and the block-scoped rules that validate a block's bundles together.
It defines the transaction representation of the
[*Tachyon pool*](draft-tachyon-shielded-protocol.md), just as ZIP 225 [^zip-0225]
defines the v5 transaction fields for the [Orchard shielded protocol](zip-0224.rst).

# Motivation

To improve scalability, the [Tachyon shielded protocol](draft-tachyon-shielded-protocol.md) introduces support for [proof aggregation](draft-tachyon-shielded-protocol.md#prooftree): the [stamps](draft-tachyon-shielded-protocol.md#stamp) of many transactions merge into one covering stamp, and covered transactions appear in a block with their stamp [stripped](#strippedstamp).
The transaction format must therefore allow a stamp to be removed without changing the transaction's identity and without invalidating any signature.
The body separates effecting data from signing and proving data. Its contents,
including tachygrams and fees, remain with their originating transaction when the
stamp is stripped.

A single format serves every transaction role: a proof stamp covers its own
bundle and lists the `txid`s of any other covered transactions. A stripped bundle
retains the data needed for signature, balance, and proof verification; block
validation resolves coverage from the proof stamps' adjunct lists.

# Requirements

* A transaction's `txid` contribution is invariant across stamping, merging, stripping, re-stamping, and fee adjustments.
* Proof-stamped and stripped forms carry identical bodies; only the state byte and stamp differ.
* Every field a validator needs for signature and balance verification is present in both states.
* Encodings are canonical: each parsed bundle has exactly one serialization.
* The proof field is length-prefixed, with its size bounded before allocation.
* Bundles with no actions are representable, under both stamp forms.

# Non-requirements

This ZIP does not independently define:

* the multiset commitment construction, which is specified in the
  [Tachyon Shielded Protocol](draft-tachyon-shielded-protocol.md#multisetcommitments) ZIP;
* the proof statement, which is specified in the Tachyon Shielded Protocol ZIP
  under [Stamp](draft-tachyon-shielded-protocol.md#stamp),
  [Proof tree](draft-tachyon-shielded-protocol.md#prooftree), and
  [Dynamic fees](draft-tachyon-shielded-protocol.md#dynamicfees);
* the tachygram accumulator, anchor semantics and validity rules, and cross-block
  duplicate-tachygram checks within the retained epoch window, which are specified
  in the Tachyon Shielded Protocol ZIP under
  [Tachygram accumulator](draft-tachyon-shielded-protocol.md#tachygramaccumulator),
  [Epochs](draft-tachyon-shielded-protocol.md#epochs), and
  [Per-stamp checks](draft-tachyon-shielded-protocol.md#per-stampchecks);
* the complete transaction digest trees or sighash algorithm. These use the
  ZIP 244 framework [^zip-0244], with Tachyon-specific
  integration still to be specified as noted under
  [Transaction digest contributions](#transactiondigestcontributions);
* P2P aggregation messages, covered-transaction discovery, or mempool and relay policy;
* the position of the bundle section within the transaction encoding, which is specified by the transaction format of the activating network upgrade.

# Specification

The specification proceeds from the wire layout of the bundle body to the rules over its fields, then to the digest constructions and multiset commitment inputs those rules use, then to the two stamp forms.
It closes with the canonical encodings of every field, the bundle's transaction-digest inputs, a consolidated summary of the bundle-scoped consensus rules, and the block-scoped rules that validate a block's bundles together.

## Placement and bundle states

The Tachyon bundle is a contiguous section of the transaction encoding, added by a Tachyon network upgrade.
The first byte of the section, `tachyonBundleState`, selects the bundle state:

| value         | state         | bundle contents                                       |
| ------------- | ------------- | ----------------------------------------------------- |
| `0b0000_0000` | non-tachyon   | no bundle                                             |
| `0b0000_0001` | proof stamp   | body, anchor, tachygram commitment, adjunct txids, proof |
| `0b0000_0010` | stripped stamp | body only                                            |
| `...`         | *reserved*    | *n/a*                                                 |

A parser MUST reject any other value of `tachyonBundleState`.
When `tachyonBundleState` is `0x00`, the Tachyon section consists of the discriminator byte alone.

The complete wire layout across the serialized states:

### Bundle Flag

| Bytes                  | Name                   | Data Type                   | Description                          |
| ---------------------- | ---------------------- | --------------------------- | ------------------------------------ |
| 1                      | `tachyonBundleState`   | `uint8`                     | `0x00`, `0x01`, or `0x02`            |

### Bundle body encoding

Present when `tachyonBundleState` is not `0x00`. Fields appear in effecting,
signing, then proving order. Signing and proving fields are authorization data.

| Section | Bytes | Name | Data Type | Description |
| ------- | ----- | ---- | --------- | ----------- |
| Effecting | 8 | `valueBalanceTachyon` | `int64` | Spends minus outputs and maximum fee |
| | 1 or 3 | `nActionsTachyon` | `compactSize` | Number of Tachyon actions, 0 to 4095 |
| | 64 * `nActionsTachyon` | `vActionsTachyon` | `byte[64][nActionsTachyon]` | Action descriptor per action |
| | 32 | `cvFeeMaxTachyon` | `byte[32]` | Maximum-fee value commitment |
| | 1, 3, or 5 | `nMemoTachyon` | `compactSize` | Byte length of memo, 0 or more |
| | `nMemoTachyon` | `vMemoTachyon` | `byte[nMemoTachyon]` | Opaque bytes |
| Signing | 64 * `nActionsTachyon` | `vActionSigsTachyon` | `byte[64][nActionsTachyon]` | Authorizing signature per action |
| | 64 | `bindingSigTachyon` | `byte[64]` | Binding signature for the bundle |
| Proving | 64 * `nActionsTachyon` | `vActionTachygramsTachyon` | `byte[32][2][nActionsTachyon]` | Tachygram pair per action |
| | 8 | `vFeeTachyon` | `uint64` | Actual fee of this bundle |
| | 32 | `cmChangeTachyon` | `byte[32]` | Fee change note commitment |

### Proof stamp encoding

Present when `tachyonBundleState` is `0x01`.

| Bytes | Name | Data Type | Description |
| ----- | ---- | --------- | ----------- |
| 32 | `anchorTachyon` | `byte[32]` | Pool state anchor |
| 32 | `cTachygrams` | `byte[32]` | Commitment to the covered tachygram union |
| 1 or 3 | `nAdjunctsTachyon` | `compactSize` | Number of other covered transactions, 0 to 8190 |
| 32 * `nAdjunctsTachyon` | `vAdjunctsTachyon` | `byte[32][nAdjunctsTachyon]` | Covered transaction txids, strictly ascending |
| 1, 3, or 5 | `sizeProofTachyon` | `compactSize` | Byte length of the Ragu proof |
| `sizeProofTachyon` | `proofTachyon` | `byte[sizeProofTachyon]` | Ragu proof, including its envelope |

### Stripped stamp encoding

When `tachyonBundleState` is `0x02`, the trailer is empty.

## Bundle body

When `tachyonBundleState` is not `0x00`, the body follows the discriminator byte, laid out as in the wire layout above.

`valueBalanceTachyon` is a two's-complement signed 64-bit integer in little-endian byte order.
`vActionsTachyon` is a sequence of `nActionsTachyon` action descriptors, each the 32-byte encoding of $\mathsf{cv}$ followed by the 32-byte encoding of $\mathsf{rk}$.
`vActionSigsTachyon` is a sequence of `nActionsTachyon` 64-byte signatures; the $i$-th signature authorizes the $i$-th descriptor.
`vActionTachygramsTachyon` contains one pair of tachygrams per action, in the same
order. All three sequences share the single count `nActionsTachyon`, so a count
mismatch between descriptors, signatures, and pairs is unrepresentable.
The descriptor sequence's order is the transaction author's choice.
The semantics of spends and outputs are described in the shielded-protocol ZIP's
[Proof tree](draft-tachyon-shielded-protocol.md#prooftree) section.

`cvFeeMaxTachyon` is the maximum-fee note's value commitment, included in the
effecting data. The actual fee, `vFeeTachyon`, and fee change note commitment,
`cmChangeTachyon`, are proving data in the body.
The fee mechanism is specified under
[Dynamic fees](draft-tachyon-shielded-protocol.md#dynamicfees) in the shielded-protocol ZIP.

The body's `vActionTachygramsTachyon` and `cmChangeTachyon` contain exactly
$2 \cdot \mathsf{nActionsTachyon} + 1$ tachygrams. A parser MUST reject any duplicate
among these values, in either bundle state. Their roles are specified under
[Tachygrams](draft-tachyon-shielded-protocol.md#tachygrams).

`vMemoTachyon` is an opaque recipient-directed payload of `nMemoTachyon` bytes; `nMemoTachyon` of $0$ encodes an absent payload.
The memo contributes to `txid` and the sighash ([Transaction digest contributions](#transactiondigestcontributions)) rather than to `auth_digest`, so it is identical across bundle states.

## Value balance and the binding signature

`valueBalanceTachyon` equals spends minus ordinary outputs minus the maximum
fee. Transaction balance and pool accounting use `valueBalanceTachyon + vFeeTachyon`:
a positive sum releases value from the *Tachyon pool*; a negative sum transfers
value into it.

Tachyon reuses Orchard's value commitments[^protocol-valuecommit] and binding-key
derivation and binding-signature construction,[^protocol-orchardbalance] applied
to the bundle's action value commitments and `valueBalanceTachyon`, with
`cvFeeMaxTachyon` included as an additional output commitment.
`bindingSigTachyon` MUST verify over the transaction sighash under the resulting
binding validating key $\mathsf{bvk}$, which is derived rather than serialized.
The actual fee does not enter this key calculation.

## Action signatures

Each action signature in `vActionSigsTachyon` MUST be a valid spend authorization signature (§ 4.15 ‘Spend Authorization Signature (Sapling and Orchard)’; RedPallas with the SpendAuth basepoint of § 5.4.7.1, for spends and outputs alike) over the transaction sighash under the corresponding action's $\mathsf{rk}$.
All of a bundle's signatures sign the same transaction-level sighash. The
Tachyon-specific inputs and pending integration with the ZIP 244 framework
[^zip-0244] are described under
[Transaction digest contributions](#transactiondigestcontributions).

## Action digests

Each action digest binds its $(\mathsf{cv}, \mathsf{rk})$ descriptor to its
unordered tachygram pair $(t_0, t_1)$ from `vActionTachygramsTachyon`.
Let $(\mathsf{cv}_x, \mathsf{cv}_y)$ and $(\mathsf{rk}_x, \mathsf{rk}_y)$ be the
affine Pallas coordinates of the action's decompressed $\mathsf{cv}$ and $\mathsf{rk}$.
Then

$$d = \mathrm{Poseidon}_{\texttt{Tachyon-ActionDg}}\bigl(
   \mathsf{cv}_x,\ \mathsf{cv}_y,\ \mathsf{rk}_x,\ \mathsf{rk}_y,\ t_0+t_1,\ t_0t_1
 \bigr).$$

The tagged Poseidon instance initializes the rate to zero and the capacity to
the Pallas base-field element whose integer value is the little-endian
interpretation of the 16-byte ASCII string `Tachyon-ActionDg`. It absorbs only
the six displayed field elements. The width, rounds, round constants, and mode
are fixed by the Ragu proof system.[^tachyon-poseidon]

If an action's $\mathsf{cv}$ or $\mathsf{rk}$ is the identity point, it has no affine coordinates, the digest is undefined, and the transaction is invalid.

## Action descriptor digests

An action's descriptor is its 64-byte encoding in `vActionsTachyon`: the 32-byte encoding of $\mathsf{cv}$ followed by the 32-byte encoding of $\mathsf{rk}$.
The descriptor digest of a sequence of actions is the BLAKE2b-256 hash, with personalization `Tachyon-Actions`, of the concatenation of their descriptors:

$$\mathsf{h} = \text{BLAKE2b-256}\bigl(
    \text{"Tachyon-Actions"},\ \mathsf{cv}_1 \| \mathsf{rk}_1 \| \cdots \| \mathsf{cv}_n \| \mathsf{rk}_n
  \bigr)$$

The digest of the empty sequence is the hash of the empty string under the same personalization.

`hActionsTachyon`, an input to the
[effecting digest contribution](#transactiondigestcontributions), is computed over
the bundle's own actions in their `vActionsTachyon` wire order. It is not carried
on the wire.

## Fee descriptors and digests

Each covered bundle contributes one fee descriptor, the pair
`(cvFeeMaxTachyon, vFeeTachyon)`. Its 40-byte encoding is the commitment's 32-byte
encoding followed by the fee's 8-byte little-endian encoding. The descriptor is
assembled from body fields, not serialized separately.

For affine Pallas coordinates $(\mathsf{cv}_{\max,x}, \mathsf{cv}_{\max,y})$ of the
maximum-fee commitment, its fee digest also binds the bundle's fee change:

$$d_{\mathrm{fee}} = \mathrm{Poseidon}_{\texttt{Tachyon-FeeDg}}\bigl(
   \mathsf{cv}_{\max,x},\ \mathsf{cv}_{\max,y},\ \mathsf{fee},\ \mathsf{cm}_{\mathrm{change}}
 \bigr).$$

Here $\mathsf{fee}$ is `vFeeTachyon` encoded as a Pallas base-field element and
$\mathsf{cm}_{\mathrm{change}}$ is `cmChangeTachyon`. The tagged Poseidon instance
is specified under [Action digests](#actiondigests), with the capacity initialized
from `Tachyon-FeeDg` padded with three zero bytes to 16 bytes.
`cvFeeMaxTachyon` MUST NOT be the identity point, which has no affine coordinates.[^tachyon-fees]

Descriptor distinctness is checked over the 40-byte pair, excluding
`cmChangeTachyon`; the fee digest binds all three values.

## Multiset commitments

Multiset commitments MUST use the construction specified in the
[Tachyon Shielded Protocol](draft-tachyon-shielded-protocol.md#multisetcommitments) ZIP.

Each proof stamp binds two commitments: `cStampActionsTachyon`, over the action
and fee digests it covers, and `cTachygrams`, over the union of the covered bodies'
tachygrams. Validators reconstruct `cStampActionsTachyon` from the covered action
descriptors and their pairs, and fee descriptors and their change commitments;
it is not serialized. The stamp carries `cTachygrams`, which validators MUST
check against the commitment reconstructed from those same pairs and change
commitments, as specified under [Block validity](#blockvalidity).

## Proof stamp

When `tachyonBundleState` is `0x01`, the proof stamp follows the body, laid out as in the wire layout above.

`vAdjunctsTachyon` lists the `txid` of every other transaction whose bundle the
stamp covers. The carrying transaction MUST NOT appear in its own list. A stamp
covering only its own bundle has an empty list. The list is strictly increasing
by bytewise comparison of the 32-byte txids, excluding duplicates.
[Block validity](#blockvalidity) specifies how these references are resolved.

`anchorTachyon` references the pool state the proof is valid against; its semantics
and the anchor-membership rule are specified in the Tachyon Shielded Protocol ZIP
under [Tachygram accumulator](draft-tachyon-shielded-protocol.md#tachygramaccumulator)
and [Per-stamp checks](draft-tachyon-shielded-protocol.md#per-stampchecks).

`cTachygrams` is the multiset commitment ([Multiset commitments](#multisetcommitments))
over the union of `vActionTachygramsTachyon` and `cmChangeTachyon` from the stamp's
own bundle and every listed adjunct. Validators MUST reconstruct this union and
confirm its commitment against the carried point.

A stamp covering $N$ actions across $B$ bundles commits to $2N+B$ tachygrams,
which MUST be distinct and number at most $8191$. Both counts include the
stamp's own bundle and every transaction it covers. A parser MUST reject
`nAdjunctsTachyon` greater than $8190$ before reading the list, and MUST reject
a proof-stamped bundle if
$2 \cdot \mathsf{nActionsTachyon} + \mathsf{nAdjunctsTachyon} + 1 > 8191$.
These bounds follow from the maximum degree of the tachygram multiset polynomial.[^tachyon-bundle]

Block-level distinctness is specified under [Block validity](#blockvalidity);
cross-block distinctness within the retained epoch window is specified in the
Tachyon Shielded Protocol ZIP under [Epochs](draft-tachyon-shielded-protocol.md#epochs)
and [Per-stamp checks](draft-tachyon-shielded-protocol.md#per-stampchecks).

`proofTachyon` is the Ragu proof.
The statement it attests to, and the base rule that a stamp proof MUST verify
against the Tachyon statement, are specified in the Tachyon Shielded Protocol ZIP
under [Stamp](draft-tachyon-shielded-protocol.md#stamp),
[Proof tree](draft-tachyon-shielded-protocol.md#prooftree), and
[Per-stamp checks](draft-tachyon-shielded-protocol.md#per-stampchecks); how that rule
is applied to a block's stamps is specified in [Block validity](#blockvalidity).

<span id="pointerstamp"></span>

## Stripped stamp

When `tachyonBundleState` is `0x02`, no fields follow the body. Stripping preserves
the entire body, including its tachygram pairs, actual fee, and fee change note
commitment. The covering transaction lists this transaction's `txid`; the
stripped bundle carries no reference to its covering transaction.
[Block validity](#blockvalidity) requires exactly one covering proof stamp.

## Canonical encodings

* Every compactSize field MUST use the minimal encoding for its value (§ 7.1 ‘Transaction Encoding and Consensus’) and MUST NOT encode a value exceeding `0x02000000`.
  A parser MUST reject any other encoding.
* `cv`, `rk`, and `cvFeeMaxTachyon` are 32-byte compressed encodings of Pallas points.
  A parser MUST reject an encoding that does not decode to a point.
  `rk` MUST decode as a RedPallas validating key (§ 5.4.7 ‘RedDSA, RedJubjub, and RedPallas’).
  The identity point is excluded by the rules in [Action digests](#actiondigests) and
  [Fee descriptors and digests](#feedescriptorsanddigests).
* `vFeeTachyon` is an unsigned 64-bit little-endian integer in zatoshis and MUST
  be at most `MAX_MONEY`.
* `anchorTachyon`, `cmChangeTachyon`, and each action tachygram are canonical little-endian encodings of Pallas base field elements; a parser MUST reject an encoding whose value is not less than the field modulus.
* `vActionsTachyon` carries no ordering requirement: the descriptors may appear in any sequence the transaction author chooses.
  The signatures and tachygram pairs follow their descriptors' positions.
* Either order within a tachygram pair is valid. Provers SHOULD place the
  numerically smaller field element first to avoid disclosing each member's role.
  The authorization digest hashes pairs in their wire order.
* `vAdjunctsTachyon` MUST be strictly increasing by bytewise comparison of its
  32-byte txids; a parser MUST reject repeated or out-of-order entries.
* `cTachygrams` is a 32-byte compressed Vesta point; a parser MUST reject an
  encoding that does not decode to a nonidentity point. Whether it commits to
  the covered bodies' tachygrams is a block-validity property ([Block validity](#blockvalidity)).
* `vMemoTachyon` is an opaque byte string at parse time; this ZIP imposes no structure on its contents.
* Signatures (`vActionSigsTachyon`, `bindingSigTachyon`) are opaque 64-byte strings at parse time; their validity is a verification-time property.
* `proofTachyon` MUST decode as a Ragu proof of exactly `sizeProofTachyon` bytes,
  including its envelope. A parser MUST reject a length exceeding the proof
  type's Ragu transport bound before allocating the frame, and MUST reject an
  invalid envelope, truncated proof, or trailing bytes within the frame.
  Block inclusion requires the `Compressed` proof form. `Uncompressed` proofs
  use the same layout for merging and lifting before inclusion.[^tachyon-bundle]

## Transaction digest contributions

The bundle contributes one leaf to each of the transaction's two digest trees
(ZIP 244 [^zip-0244]). The constructions below specify these contributions.[^tachyon-digests]

TODO: Specify their integration into the activating transaction format's digest
trees and sighash algorithm, including final transaction-leaf personalizations.

In this section, $H_p(m)$ denotes BLAKE2b-256 with personalization $p$ and input
$m$. Personalizations are zero-padded to 16 bytes. Integers use their little-endian
wire encodings; points and field elements use their canonical wire encodings.

The effecting contribution (to `txid` and the sighash) is

$$H_{\texttt{ZTxIdTachyonHash}}\bigl(
\mathsf{hActionsTachyon} \| \mathsf{valueBalanceTachyon} \|
\mathsf{cvFeeMaxTachyon} \| \mathsf{hMemoTachyon}\bigr),$$

where `hActionsTachyon` is the digest of the bundle's own action descriptors in
wire order ([Action descriptor digests](#actiondescriptordigests)) and
$\mathsf{hMemoTachyon} = H_{\texttt{Tachyon-Memo}}(\mathsf{vMemoTachyon})$.
Tachygrams, the actual fee, and the stamp are excluded, so stamping, merging,
stripping, re-stamping, and fee adjustments preserve this contribution.

The authorizing contribution (to `auth_digest`) is

$$H_{\texttt{ZTxAuthTachyHash}}\bigl(
\mathsf{tachyonBundleState} \|
\mathop{\|}_i (\mathsf{sig}_i \| t_{i,0} \| t_{i,1}) \|
\mathrm{LE64}(\mathsf{vFeeTachyon}) \| \mathsf{cmChangeTachyon} \|
\mathsf{bindingSigTachyon} \| \mathsf{hStampTachyon}\bigr).$$

The concatenation contains one 128-byte signature-and-pair record per action,
in action order, preserving each pair's wire order. This digest ordering differs
from the body's wire ordering; neither `nActionsTachyon` nor a pair count is hashed.

For a proof stamp:

$$\begin{aligned}
\mathsf{hStampProofTachyon} &= H_{\texttt{Tachyon-Proof}}\bigl(
\mathsf{anchorTachyon} \| \mathsf{cTachygrams} \| \mathsf{proofTachyon}\bigr), \\
\mathsf{hStampTachyon} &= H_{\texttt{Tachyon-Stamp}}\bigl(
\mathsf{vAdjunctsTachyon} \| \mathsf{hStampProofTachyon}\bigr).
\end{aligned}$$

The proof bytes include the Ragu envelope, excluding `sizeProofTachyon`.
Adjunct txids are concatenated in wire order, excluding `nAdjunctsTachyon`.
For a stripped stamp, `hStampTachyon` is $H_{\texttt{Tachyon-Stamp}}$ of the empty
string. Changing the fee, fee change, pair order, or stamp changes `auth_digest`
and `wtxid`. A stripped bundle's `wtxid` is independent of its covering transaction.

A transaction with no Tachyon bundle contributes the hash of the empty string
under each transaction-leaf personalization. This differs from a present bundle
with no actions, which still contributes its body and stamp digest.

## Bundle validity

The rules owned by this ZIP, applying to a single transaction's bundle:

1. `tachyonBundleState` MUST be `0x00`, `0x01`, or `0x02`.
2. Every compactSize MUST be minimally encoded and MUST NOT exceed `0x02000000`.
3. Every point and field-element encoding MUST be canonical, and every sequence in canonical order, as specified in [Canonical encodings](#canonicalencodings).
4. An action's $\mathsf{cv}$ and $\mathsf{rk}$, and `cvFeeMaxTachyon`, MUST NOT be the identity point.
5. Every action signature MUST verify over the transaction sighash under its action's $\mathsf{rk}$.
6. `bindingSigTachyon` MUST verify over the transaction sighash under the derived $\mathsf{bvk}$.
7. `valueBalanceTachyon` MUST be in the range $-\mathrm{MAX\_MONEY}$ to $\mathrm{MAX\_MONEY}$ inclusive.
8. A bundle with no actions MUST have `valueBalanceTachyon` and `vFeeTachyon` equal to $0$.
9. A proof stamp's `proofTachyon` MUST satisfy the length, envelope, and proof-form rules under [Canonical encodings](#canonicalencodings).
10. The tachygrams in a bundle's action pairs and fee change commitment MUST be distinct.
11. A proof stamp's adjunct list MUST be strictly increasing, MUST NOT include its own transaction's `txid`, and MUST contain at most $8190$ entries.
12. `vFeeTachyon` MUST be in the range $0$ to $\mathrm{MAX\_MONEY}$ inclusive.
13. `nActionsTachyon` MUST be at most $4095$. A proof-stamped bundle MUST also satisfy $2 \cdot \mathsf{nActionsTachyon} + \mathsf{nAdjunctsTachyon} + 1 \leq 8191$.

Rules outside the scope of this ZIP are enumerated in [Non-requirements](#non-requirements).

## Block validity

The rules owned by this ZIP that constrain a block's Tachyon bundles together.
A validator enforces them fail-fast, in this order:

1. **Tachygram uniqueness.** All tachygrams in a block MUST be distinct.
   Collect `vActionTachygramsTachyon` and `cmChangeTachyon` from every bundle body,
   whether proof-stamped or stripped, and reject any duplicate. Reuse across
   blocks within the retained epoch window is governed by the Tachyon Shielded
   Protocol ZIP's [Per-stamp checks](draft-tachyon-shielded-protocol.md#per-stampchecks).
2. **Proof coverage.** Every `txid` in a proof stamp's `vAdjunctsTachyon` MUST
   identify exactly one transaction in the same block, whose Tachyon bundle
   MUST be stripped. Every stripped bundle MUST be listed by exactly one proof
   stamp, including bundles with no actions. A validator MUST check for unlisted
   stripped bundles across the entire block; traversing adjunct lists alone is
   insufficient.[^tachyon-aggregation]
3. **Covered descriptors and tachygram bound.** For each proof stamp, collect its
   own bundle and the stripped bundles listed by its `vAdjunctsTachyon`. Their
   action descriptors MUST be mutually distinct, as MUST their fee descriptors.
   For $N$ covered actions and $B$ covered bundles, $2N+B$ MUST NOT exceed $8191$.
   Reject repeated descriptors or an exceeded bound before curve arithmetic.
4. **Tachygram commitment.** Reconstruct the multiset commitment over the action
   pairs and fee change commitments from those covered bodies
   ([Multiset commitments](#multisetcommitments)). It MUST equal the stamp's
   carried `cTachygrams`.
5. **Proof verification.** Every proof stamp MUST verify against its anchor,
   the `cTachygrams` confirmed by rule 4, and `cStampActionsTachyon` reconstructed
   from the covered action and fee digests ([Action digests](#actiondigests),
   [Fee descriptors and digests](#feedescriptorsanddigests),
   [Multiset commitments](#multisetcommitments)). The base requirement that a
   proof verifies the Tachyon statement is specified in the Tachyon Shielded
   Protocol ZIP's [Per-stamp checks](draft-tachyon-shielded-protocol.md#per-stampchecks);
   this rule applies it per stamp within a block.

# Rationale

This subsection is non-normative.

## The stamp is excluded from the txid contribution

The effecting contribution excludes the stamp, so stamping, merging, stripping, re-stamping, and fee adjustments preserve a transaction's logical identity.
This allows proof aggregation without changing the transaction its author signed:
a stripped transaction retains the same body and signatures.

## Zero-action balance

The v5 analogue omits `valueBalanceOrchard` when the action count is zero and defines its value as zero in that case.
Tachyon gates field presence on the bundle state, not the action count: the non-tachyon state omits the field, while a present bundle always encodes `valueBalanceTachyon`.
A present bundle with no actions therefore still carries the field, so the zero balance is stated as an explicit rule.
Together with the binding signature, this requires a zero maximum fee; the
[dynamic fee mechanism](draft-tachyon-shielded-protocol.md#dynamicfees) also
requires a zero actual fee. Its fee descriptor and fee change note commitment
remain part of the covering proof.

## Identity points are invalid

The action and fee digests hash affine coordinates, which the identity point lacks.
Excluding it also rejects a degenerate verification key and a degenerate value commitment.

## Adjunct txids are sorted

Sorting `vAdjunctsTachyon` gives each adjunct set one wire encoding and excludes
repeated entries. The list identifies the covered bundles directly; validators
reconstruct the proof inputs from those bodies.

## Covered actions are required to be distinct

Two identical descriptors carry an identical $\mathsf{rk}$, so the same signature would authorize both if they sign the same transaction sighash.
Multiset commitments preserve multiplicity and do not establish distinctness.
[Block validity](#blockvalidity) rule 3 therefore rejects repeated action or fee
descriptors before commitment reconstruction and proof verification.

## Tachygram pairs are unordered

The action digest binds a pair through its sum and product, so swapping its two
members preserves the proof statement. Either wire order is accepted, while the
authorization digest records the order actually transmitted. A prover sorts
each pair numerically to avoid revealing which member is a commitment,
nullifier, or padding value.

## Deterministic multiset commitments

The committed multisets are public data, so a hiding commitment is unnecessary; determinism is what lets any party recompute a commitment from the data it covers.

## Length-prefixed proof

A length prefix frames the Ragu proof and its envelope. Bounding that length
before allocation limits untrusted input; exact decoding rejects extra bytes
within the frame.

## Adjuncts are referenced by txid

An aggregate lists its adjuncts, so lifting or rerandomizing the aggregate does
not require rewriting them. A `txid` identifies effecting data; the covering
proof additionally binds each adjunct's actual fee, fee change, and unordered
action pairs. A body with different values fails verification.

# Security Implications

This subsection is non-normative.

## Canonical encodings and ordering

The canonical-encoding rules restrict field encodings and adjunct-list order;
they do not imply that a transaction has only one valid proof, set of signatures,
or ordering of each tachygram pair. Swapping a pair changes `auth_digest` and
`wtxid` without changing its proof statement.
Reordering distinct action descriptors changes `hActionsTachyon`, and therefore the sighash every signature covers, so existing signatures do not authorize the reordered transaction.
Leaving action order to the author's choice is the same property Sapling and Orchard rely on for their own unordered spend, output, and action arrays.
Authorization-form changes (re-stamping, stripping) produce distinct bundles by design and are reflected in `auth_digest` and `wtxid` (ZIP 239 [^zip-0239]).

## Duplicate actions are gated after parsing, not by the parser

The parser enforces no distinctness over `vActionsTachyon`, so a bundle carrying a repeated action descriptor parses.
Each descriptor's $\mathsf{cv}$ enters the binding sum, so action multiplicity is load-bearing and the parser cannot deduplicate without changing the asserted balance; the action digests enter the stamp's `cStampActionsTachyon` alongside its covered fee digests.

[Block validity](#blockvalidity) rule 3 explicitly rejects repeated action or fee descriptors in a proof stamp's combined coverage before commitment reconstruction.
This includes duplicates within one bundle and across bundles covered by the same stamp.
The multiset commitment preserves repeated roots; it does not itself prohibit them.
A parser must therefore preserve multiplicity rather than silently deduplicate the actions.

Repeated tachygrams in a body's action pairs or fee change commitment violate
the explicit distinctness rule, which the parser enforces directly
([Bundle validity](#bundlevalidity) rule 10).

## Complete proof coverage

A stripped bundle retains all its actions and balance data but carries no proof.
An unlisted stripped bundle would therefore leave those values unproven. Block
validation must check all stripped bundles for coverage, not only those reached
through proof stamps' adjunct lists. A listed txid does not itself establish
proof coverage: the reconstructed commitments and proof must also verify.

## Balance consistency is enforced by the binding property

A valid binding signature establishes that `valueBalanceTachyon` equals the net value committed by the actions' $\mathsf{cv}$ together with `cvFeeMaxTachyon`, by the same binding argument as Orchard (§ 4.14).

## Signatures survive stripping

All signatures cover the transaction sighash, which incorporates only effecting data.
A miner stripping a proof stamp changes no signed data, so aggregation does not invalidate signatures.
A signature authorizes only the sighash it signs; using an action in a transaction with different effecting data requires a new signature over that transaction's sighash.

<div class="note"></div>

Parse validity is not spend validity.
A bundle that parses and whose signatures verify is not thereby valid to spend;
the [Block validity](#blockvalidity) rules and the shielded protocol's
[Consensus rules](draft-tachyon-shielded-protocol.md#consensusrules) also apply.
Implementers and auditors should not assume the rules in this ZIP alone establish those properties.


# Privacy Implications

This subsection is non-normative.

## Public data

$\mathsf{cv}$ is a hiding commitment to the action's value; `cvFeeMaxTachyon` hides the maximum fee.
The privacy of the underlying notes and keys depends on the shielded protocol; see
[Keys and addresses](draft-tachyon-shielded-protocol.md#keysandaddresses) and
[Tachygrams](draft-tachyon-shielded-protocol.md#tachygrams). Payload confidentiality
and note discovery depend on the payment protocol, as discussed under
[Privacy and Security Implications](draft-tachyon-shielded-protocol.md#privacyandsecurityimplications).
The action count, value balance, actual fee, each action's tachygram pair, and
the fee change note commitment are public. The pair is associated with its
action, but its members are untyped. The separate fee change field identifies
that tachygram as a note commitment.

Comparing versions of a transaction re-proved for different epochs reveals which
action pairs change (spends) and which stay fixed (outputs). A fee adjustment
alone leaves the action pairs unchanged.[^tachyon-tachygrams] A delegate that
knows a note's nullifier window can recognize its spending action from the
published pair.[^tachyon-nullifiers]

Aggregation does not hide in-block coverage: proof stamps publicly list their
adjuncts' txids, including for bundles with no actions (see
[Block validity](#blockvalidity)).

# Deployment

This ZIP will be deployed with [NuTachyon](draft-tachyon-nutachyon-upgrade.md).

# Reference implementation

The `zcash_tachyon` crate in the
[Tachyon repository](https://github.com/tachyon-zcash/tachyon) provides the bundle
codec, commitment and digest calculations, and signature verification.
Experimental transaction-format, value-accounting, and transaction-digest
integration is provided by
[zakura-core/common PR #178](https://github.com/zakura-core/common/pull/178).
Experimental full-node integration, including bundle and block validation, is
provided by [Zakura PR #795](https://github.com/zakura-core/zakura/pull/795).

# References

[^BCP14]: [Information on BCP 14: "RFC 2119: Key words for use in RFCs to Indicate Requirement Levels" and "RFC 8174: Ambiguity of Uppercase vs Lowercase in RFC 2119 Key Words"](https://www.rfc-editor.org/info/bcp14)

[^protocol]: [Zcash Protocol Specification](protocol/protocol.pdf)

[^protocol-valuecommit]: [Zcash Protocol Specification, Version v2026.7.0-202-gafa086 [NU6.2]. Section 5.4.8.3: Homomorphic Pedersen commitments (Sapling and Orchard)](protocol/protocol.pdf#concretehomomorphiccommit)

[^protocol-orchardbalance]: [Zcash Protocol Specification, Version v2026.7.0-202-gafa086 [NU6.2]. Section 4.14: Balance and Binding Signature (Orchard)](protocol/protocol.pdf#orchardbalance)

[^tachyon-fees]: [The Tachyon Book: Dynamic Fee](https://github.com/tachyon-zcash/tachyon/blob/9d9af51a9c4e8ba1e19f5485eafe066e46137be9/book/src/fee.md)

[^tachyon-tachygrams]: [The Tachyon Book: Tachygrams](https://github.com/tachyon-zcash/tachyon/blob/9d9af51a9c4e8ba1e19f5485eafe066e46137be9/book/src/tachygrams.md)

[^tachyon-nullifiers]: [The Tachyon Book: Nullifiers](https://github.com/tachyon-zcash/tachyon/blob/9d9af51a9c4e8ba1e19f5485eafe066e46137be9/book/src/nullifiers.md)

[^tachyon-bundle]: [The Tachyon Book: Bundles](https://github.com/tachyon-zcash/tachyon/blob/9d9af51a9c4e8ba1e19f5485eafe066e46137be9/book/src/bundle.md)

[^tachyon-digests]: [The Tachyon Book: Transaction Identifiers](https://github.com/tachyon-zcash/tachyon/blob/9d9af51a9c4e8ba1e19f5485eafe066e46137be9/book/src/transaction-identifiers.md)

[^tachyon-poseidon]: [Tachyon Poseidon digest implementation](https://github.com/tachyon-zcash/tachyon/blob/9d9af51a9c4e8ba1e19f5485eafe066e46137be9/crates/tachyon/src/digest/poseidon.rs)

[^tachyon-aggregation]: [The Tachyon Book: Aggregation, Block rules](https://github.com/tachyon-zcash/tachyon/blob/9d9af51a9c4e8ba1e19f5485eafe066e46137be9/book/src/aggregation.md#block-rules)

[^zip-0200]: [ZIP 200: Network Upgrade Mechanism](zip-0200.rst)

[^zip-0225]: [ZIP 225: Version 5 Transaction Format](zip-0225.rst)

[^zip-0239]: [ZIP 239: Relay of Version 5 Transactions](zip-0239.rst)

[^zip-0244]: [ZIP 244: Transaction Identifier Non-Malleability](zip-0244.rst)

[^zip-0248]: [ZIP 248: Extensible Transaction Format](zip-0248.rst)
