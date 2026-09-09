<pre>
  DIP: pasta-compact-quorum-proofs
  Title: Compact Quorum Proof Chains for Trustless Platform Verification
  Author(s): PastaPastaPasta
  Special-Thanks:
  Comments-Summary: No comments yet.
  Status: Draft
  Type: Standard
  Created: 2026-01-17
  License: MIT License
</pre>

## Table of Contents

1. [Abstract](#abstract)
1. [Motivation](#motivation)
1. [Protocol Foundations](#protocol-foundations)
1. [Trust Model](#trust-model)
1. [Wire Format](#wire-format)
1. [Verification](#verification)
1. [Construction and Serving](#construction-and-serving)
1. [SDK Integration](#sdk-integration)
1. [Size and Resource Limits](#size-and-resource-limits)
1. [Security Considerations](#security-considerations)
1. [Copyright](#copyright)

## Abstract

This proposal authenticates current Dash Core quorum keys and EvoNode records from
an application-supplied trusted snapshot. A relay provides ordinary ChainLock
certificates, quorum mining transactions, and Merkle paths. The SDK verifies them
locally before verifying Platform responses. Relays supply evidence and do not
supply trusted keys. The proof uses existing Dash blocks, commitments, signatures,
and consensus rules.

Each handoff authenticates the next quorum through its mining transaction.
When the mining block lacks a usable ChainLock, a later certificate authenticates
that block through consecutive X11 headers.

## Motivation

An SDK can distribute a small, fixed Core snapshot and network addresses with its
release, then acquire the evidence needed to authenticate newer Platform quorum
keys. The intended history window is three to twelve months. The proof and the
SDK verifier both contribute to download cost, so they must be measured together.

## Protocol Foundations

[DIP-0004](dip-0004.md) commits simplified masternode lists in coinbase transactions.
[DIP-0006](dip-0006.md) defines LLMQ commitments and
[DIP-0008](dip-0008.md) defines ChainLocks.

A mining transaction contains the complete next quorum commitment. A handoff
authenticates that transaction with a Merkle path to a ChainLock-authenticated
block. The final coinbase supplies the quorum and masternode roots used to
authenticate the requested records.

## Trust Model

The application independently fixes a snapshot containing:

* Network (0 mainnet, 1 testnet).
* Core height and block hash.
* Simplified masternode-list Merkle root.
* Active quorum-list Merkle root.

A response MUST exactly match this snapshot. A relay-provided snapshot MUST NOT
be promoted to trusted configuration merely because a proof is internally valid.
A successfully verified target can serve as the next session checkpoint.

This proposal defines mainnet and testnet snapshots and quorum parameters.
Devnet and regtest require application-supplied trust configuration.

The design assumes historically authenticated ChainLock quorums do not sign false
certificates, including after leaving the active set. It proves a sequence of
statements by authenticated quorum keys. It does **not** independently reconstruct
DKG, full Core consensus, or the exact active signing-quorum selection for each
certificate. Membership of a key is not proof of its current signing authority.
These are deliberate constraints of this compact certificate trust model.

## Wire Format

All proof framing integers are unsigned little-endian fixed-width integers.
Hashes are 32 bytes in Core serialization order, reversed from RPC display hex.
Nested transactions and commitments use their existing canonical Core consensus
serialization, including CompactSize where consensus requires it. There is no
protobuf or general-purpose object encoding on the proof wire.

A `blob` is `length:u32 || bytes[length]`. A `path` is:

```text
index:u32 | leaf_count:u32 | sibling_count:u8 | siblings[32]...
```

A `certificate` is exactly 180 bytes:

```text
height:u32 | core_block_header[80] | Basic_BLS_signature[96]
```

The main proof is:

```text
magic[8] = ASCII "DASHNC02"
snapshot {
  network:u8 | height:u32 | block_hash[32]
  masternode_root[32] | quorum_root[32]
}
seed_commitment:blob | seed_membership:path
handoff_count:u16
handoffs[handoff_count] {
  certificate
  mining_transaction:blob | transaction_membership:path
  ancestor_count:u16 | ancestor_headers[80]...
}
final_certificate
final_coinbase:blob | coinbase_membership:path
```

The seed is the complete final quorum commitment, including its vector hash and
**both** embedded signatures. Its double-SHA256 hash is opened in the snapshot's
quorum root. Embedded commitment signatures are included in the authenticated
serialization; this verifier does not re-execute their DKG validation.

Ancestor headers are ordered oldest first: mining block, then its descendants,
ending at the certificate's parent. Zero ancestors means the certificate signs
the mining block itself. The mining height equals certificate height minus
ancestor count. There is no independent relay-selected mining height.

The HTTP bootstrap envelope adds authenticated consensus records:

```text
proof:blob
record_count:u8
records[record_count] {
  kind:u8                  # 0 quorum commitment; 1 simplified masternode entry
  consensus_leaf:blob
  membership:path
}
```

A quorum leaf is the full commitment. A masternode leaf is exactly the
`CSimplifiedMNListEntry::CalcHash` preimage, excluding the network-only version
prefix. The decoder must consume the complete supported canonical serialization;
ambiguous or unsupported masternode encodings are rejected.

## Verification

1. Enforce all framing limits before allocation. Reject truncation, trailing
   bytes, unknown tags, and a magic value other than `DASHNC02`.
2. Require exact equality with the application's trusted snapshot. The snapshot
   MUST be after v20 activation on mainnet or testnet; the proof uses Basic BLS
   signatures and v3 coinbase payloads.
3. Parse the seed commitment canonically, require a nonzero subgroup-valid public
   key, and verify its membership in the snapshot quorum root.
4. For each handoff, require a strictly increasing certificate height and the
   network's ChainLock quorum type (2 mainnet, 1 testnet). Verify its Basic BLS
   signature using the current key and Dash's existing ChainLock request/signing
   hash construction, including the certificate height, quorum identity, and X11
   block hash.
5. Starting at that signed header, check every `hashPrevBlock` against the X11
   hash of the preceding supplied header. Verify the complete mining transaction
   against the oldest header's transaction root. Transaction index must be
   nonzero. Require a canonical v3 quorum-commitment transaction with no inputs,
   outputs, or locktime, payload version 1, and the derived mining height.
6. Parse its full non-null commitment and install the authenticated next key.
   Reject a handoff to the same quorum identity. The mining block may precede
   the initial snapshot; certificate heights still advance beyond it.
7. Verify the final certificate with the last key. Open transaction index zero,
   parse its complete v3 coinbase, and require coinbase payload height to equal
   the signed height. Extract both final roots.
8. Enforce the caller's minimum target height. Verify every requested record's
   double-SHA256 leaf hash against its corresponding final root. Require the
   requested Platform quorum type and hash to match the authenticated commitment.
9. Publish the new state, key, and eligible EvoNode endpoints only after the entire
   envelope succeeds. Then verify the Platform response signature and GroveDB
   proof before returning application data or advancing Platform freshness state.

Merkle verification consumes exactly the tree depth implied by leaf count. An
odd final node must use its own hash as the duplicate sibling. Equal siblings at
non-duplicate positions are rejected. Transaction and record leaves of exactly
64 bytes are rejected to prevent interpreting an internal tree node as a leaf.

## Construction and Serving

An unpruned Core node opts in with `-quorumproofindex`. Startup scans historical
blocks to archive coinbase-carried ChainLocks and quorum mining transaction
paths. Missing history causes indexing to fail explicitly. Disconnect handling
tracks the carrier block, preserving evidence from an earlier carrier when a
later repeated certificate disconnects.

Construction works backwards from the requested target signer to a quorum present
in the initial snapshot. For each needed quorum, the node locates its mining
transaction and a usable certificate at or after mining. Searching nearby
certificates minimizes serialized bytes per height advanced; the search can
expand when a nearby ChainLock is unavailable. This heuristic is not part of
verification or a claim of global minimum size. Construction fails explicitly if
history, a bridge, or the resource budget is unavailable. Multiple bounded
requests can advance a checkpoint over longer gaps.

The Core RPC is:

```text
getquorumproofchain checkpoint_hash height=0 quorum_hash="" llmq_type=0 node_count=4
```

`height=0` chooses the latest archived certificate within the search budget. A
positive height is a minimum: the node searches for a certificate at or above
both it and snapshot height plus one. `quorum_hash` and `llmq_type` request one
quorum opening; `node_count` requests zero through fifteen eligible EvoNodes.
The result contains `proof_hex`, `bootstrap_hex`, and `target`. The bootstrap
field is empty if no records were requested. Generation supports mainnet/testnet.

```text
verifyquorumproofchain checkpoint_object proof_hex minimum_height=0
```

The verification RPC takes all independently trusted snapshot fields and returns
`valid` plus either the authenticated `target` or an `error`. It does not consult
RPC metadata to obtain trust roots.

DAPI and quorum servers expose the same relay interface:

```http
POST /proofs
Content-Type: application/json

{"checkpoint":"<RPC block hash>","height":1549547,
 "quorumHash":"<RPC quorum hash>","llmqType":6,"nodeCount":4}
```

Success is the binary bootstrap envelope with content type
`application/octet-stream`; HTTP gzip compression is permitted. Servers bound
request sizes, concurrent Core workers, cached bytes, and cache lifetime.
A timeout does not release a worker permit while its blocking RPC is still
running. Failure returns an HTTP error, never trusted fallback keys.

## SDK Integration

Mainnet/testnet network builders use the verified provider by default. Release
snapshots and untrusted seed addresses are embedded. Proof sources can be seeded
EvoNodes, quorum servers, or an explicitly supplied list of either. Authenticated
EvoNode records add connection candidates; addresses themselves never confer
signing authority.

Verified SDK operation requires reachable proof-serving endpoints backed by
index-enabled Core nodes.

The synchronous Platform verifier reports a typed missing-quorum condition. The
SDK asynchronously obtains a bootstrap proof, verifies it, then repeats the
complete Platform verification. This applies to reads and transaction results,
and works without synchronous network calls in browser verification code.

Applications can explicitly select trusted mode or supply their own context
provider. Trusted mode obtains quorum keys from its configured provider and skips
the additional Core bootstrap proof download and verification; Platform response
proof verification remains separately configurable. Failed verified mode MUST NOT
silently become trusted mode.

## Size and Resource Limits

| Item | Maximum |
| --- | ---: |
| Decoded proof or bootstrap HTTP response | 1,048,576 bytes |
| Certificates, including final certificate | 4,096 |
| Ancestor headers across the whole proof | 4,096 |
| Merkle leaf count | 100,000 |
| Merkle siblings | 17 |
| Transaction blob | 100,000 bytes |
| Seed commitment blob | 1,024 bytes |
| Bootstrap records | 16 |
| Record leaf | 4,096 bytes |

These limits bound individual requests, not the duration of history. Certificate
availability and quorum cadence determine achievable history per request.

Real testnet history encoded by the reference implementation measured
85,827 / 159,536 / 314,357 gzip bytes for 90 / 180 / 366 days respectively.
The shared short cross-implementation fixture is 3,469 raw proof bytes, or 4,506
raw bytes with one quorum and one EvoNode opening. These are testnet observations,
not mainnet measurements or worst-case guarantees. Final record openings add to
the history-only figures. HTTP compression is a transport optimization and does
not change verification.

The release requirement is less than 500,000 additional SDK download bytes.
A standalone verifier artifact cannot establish the integrated SDK delta; compare
matching release targets and compression settings before shipping a release.

## Security Considerations

An attacker controlling all relays can withhold evidence, replay sufficiently
recent valid evidence, or exhaust a client's bounded request budget. Successful
verification establishes authenticity under the trust model, not that the target
is the globally newest block. Platform signed-time/height freshness policy and
caller minimum heights are required. Unauthenticated metadata must not advance a
freshness ratchet.

The design does not protect against compromise of enough historical quorum keys
to forge this certificate chain. Stronger guarantees require a stronger trust
model or additional consensus evidence and have different size costs.

Snapshots are release trust material. Their hashes and roots require independent
release verification and network binding. Updating a snapshot from an unverified
HTTP response defeats the design. Persistent caches, if implemented, require the
same provenance and integrity protections as their original trust configuration.

No headers are treated as authenticated merely because a matching quorum key is
known. Every accepted statement is covered by the certificate chain under the
historical quorum honesty assumption above. Full state validity and exact signer
eligibility are deliberately outside this proof's statement.

## Copyright

Copyright (c) 2026 PastaPastaPasta. Licensed under the MIT License.
