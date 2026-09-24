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

This document proposes the Tachyon shielded protocol, which defines a new shielded pool with various scalability improvements. Tachyon enables validators to prune nullifier state, and supports aggregation of ZK proofs.

# Motivation

The Orchard shielded protocol specified by ZIP-224 introduced a proof system designed around a curve cycle, with two main motivations:

- Removing the system's reliance on a "trusted setup", both to harden the trust assumptions of the shielded protocol and to remove the coordination complexity of the setup process.
- Introduce the possibility of integrating known scaling techniques based on recursive zero-knowledge proofs in a future upgrade.

While the first goal was achieved, the Orchard shielded protocol does not utilize Halo 2's recursion compatibility for a scalable protocol.

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
- By requiring users to privately obtain and supply a recursive zero-knowledge proof of their notes spendability along with their transaction, nullifier state may be organized into _epochs_ and periodically pruned by validators.
- By allowing proofs from multiple transactions to be aggregated into a single proof, storage and verification costs can be amortized for proof data.

# Specification

The Tachyon protocol MUST be implemented as specified in the Zcash Protocol
Specification.[^protocol-tachyon]


## Cryptographic primitives


### Curves and fields


### Commitment schemes


### Signature schemes


### Proving system

Unlike Orchard's usage of Halo 2 with PLONKish arithmetization, Tachyon uses the Ragu proving system,[^ragu] with R1CS-like arithmetization inspired by Bootle16[^bootle16] and efficient recursion via split accumulation.[^bclms21]


## Key hierarchy and addresses


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

[^zip-0225]: [ZIP 225: Version 5 Transaction Format](zip-0225.rst)

[^ragu]: [Ragu proving system](https://tachyon.z.cash/ragu/)

[^bootle16]: Jonathan Bootle, Andrea Cerulli, Pyrros Chaidos, Jens Groth, and Christophe Petit. [Efficient Zero-Knowledge Arguments for Arithmetic Circuits in the Discrete Log Setting](https://eprint.iacr.org/2016/263). Cryptology ePrint Archive, Paper 2016/263, 2016.

[^bclms21]: Benedikt Bünz, Alessandro Chiesa, William Lin, Pratyush Mishra, and Nicholas Spooner. [Proof-Carrying Data without Succinct Arguments](https://eprint.iacr.org/2020/1618). Cryptology ePrint Archive, Paper 2020/1618, 2021.
