---
name: supra-verify-onchain-proof
description: Verify that a Supra event was really emitted, or that a transaction was really included, using the Merkle proof and inclusion-certificate endpoints — so an agent can trust chain data without trusting the node that served it.
api: Supra RPC Node API
base_url: https://rpc-mainnet.supra.com
operations:
  - events_by_type_v4
  - events_by_type_v3
  - event_emission_proof_v4
  - proofs_event_emission_proof_v4
  - proofs_events_v4
  - transactions_inclusion_certificates_by_block_height_v4
  - transaction_by_hash_v4
  - committee_authorization_v4
  - committees_v4
generated: '2026-08-29'
method: generated
source: openapi/supra-rpc-node-openapi.yml — every operationId is grepped from the served contract.
---

# Verify Supra chain data instead of trusting it

Most blockchain REST APIs hand you JSON and ask you to believe it. Supra's v4 surface does
not: inclusion and emission proofs are first-class operations, so an agent acting on chain
data can check the data against a certified Merkle root. If you are automating anything with
money attached, use this path rather than the plain read endpoints.

## 1. Find the events

    GET /rpc/v4/events/{event_type}       # events_by_type_v4

`event_type` is a fully qualified Move event type, e.g.
`0x1::coin::DepositEvent`. URL-encode it. Returns `EventsV4` — `EventWithContextV4` items
carrying the `EventV2`, the `transaction_hash` that emitted it, and optionally `proofs`
(`TransactionInclusionAndEventEmissionProof`).

Paginate with `count` / `start`, and read `x-supra-cursor` for the next page.

## 2. Get the emission proof

For one event:

    GET /rpc/v4/events/{event_hash}/transaction/{transaction_hash}         # event_emission_proof_v4
    GET /rpc/v4/proofs/events/{event_hash}/transaction/{transaction_hash}  # proofs_event_emission_proof_v4

For many at once:

    POST /rpc/v4/proofs/events            # proofs_events_v4

Body is an `EventEmissionProofsBatchRequest`. Per the contract this returns "a batch of
event-emission proofs anchored to a single inclusion certificate" — which is the point:
one certificate covers the whole batch, so verifying N events costs one signature check,
not N.

A batch that is too large returns **400** ("batch too large"). The contract does not publish
the cap, so discover it by backing off, and cache the working size.

## 3. Get the inclusion certificate

    GET /rpc/v4/transactions/certificates  # transactions_inclusion_certificates_by_block_height_v4

Returns `TransactionsInclusionCertificates` — `Certificate_TransactionsInclusionCertificationData`
items anchored at a block height.

`404` here means "No inclusion certificate is currently available on this node" — it may
become available. Poll with backoff.

## 4. Check who signed it

    GET /rpc/v4/consensus/committees/{epoch}                # committees_v4
    GET /rpc/v4/consensus/committee_authorization/{epoch}   # committee_authorization_v4

A certificate is only worth as much as the committee that signed it. These return the
`AuthorizedCommittee` and `CommitteeAuthorization` for an epoch, including
`ValidatorPublicKeys`, so you can verify the threshold signature yourself.

`400` means the epoch does not exist; `410` means it has been pruned.

## 5. Verify the shape

`TransactionInclusionProof` carries:

- `proof` — an `AccumulatorProof`
- `leaf_hash_value`
- `merkle_root_hash_value`
- `events_tree_merkle_root_hash_value`
- `events_leaves_hash_values`
- `certified_at_height`

Recompute the root from the leaf and the accumulator path, and compare against the
certified root. Request the response as `application/x-bcs` rather than JSON where you can:
BCS is the canonical serialization the hashes were computed over, so hashing the JSON
rendering will not match.

## Failure modes that matter

| Status | Meaning | What to do |
|---|---|---|
| 400 | Malformed hash, or proof batch too large | Fix input / halve the batch |
| 404 | Certificate not yet available on this node | Poll with backoff |
| 410 | Epoch or range pruned | Permanent here — use an archival node |
| 503 | Archive still initializing | Retry with backoff |

Never treat an unverified read as verified because the proof endpoint returned 404. Absence
of a proof is not evidence.
