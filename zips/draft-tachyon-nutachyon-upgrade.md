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


# Abstract


# Motivation


# Privacy Implications


# Requirements


# Non-requirements


# Specification


## Network upgrade constants


### Consensus branch ID


### Activation heights


### Minimum network protocol versions


## Network upgrade changes


### Tachyon shielded protocol

See the [Tachyon Shielded Protocol](draft-tachyon-shielded-protocol.md#specification)
ZIP for cryptographic constructions, proof statements, epochs, and pool-state rules.


### Tachyon bundle format

See the [Tachyon Bundle / Aggregate Transaction Format](draft-tachyon-bundle-format.md#specification)
ZIP for bundle encodings and transaction-digest inputs.


### Transaction format


### Transaction identifiers and signature digests


### Consensus rules

See the shielded-protocol ZIP's
[Consensus rules](draft-tachyon-shielded-protocol.md#consensusrules) and the
bundle-format ZIP's [Bundle validity](draft-tachyon-bundle-format.md#bundlevalidity)
and [Block validity](draft-tachyon-bundle-format.md#blockvalidity) sections.


### Network behavior

See the [Tachyon Aggregator Protocol](draft-tachyon-aggregation-protocol.md#specification)
ZIP for the aggregation lifecycle, including
[Transaction identifiers and P2P relay](draft-tachyon-aggregation-protocol.md#transactionidentifiersandp2prelay).


## Backward compatibility


# Rationale


# Security Implications


# Deployment


# Reference implementation


# Open issues


# References
