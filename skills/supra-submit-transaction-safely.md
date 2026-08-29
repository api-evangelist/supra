---
name: supra-submit-transaction-safely
description: Submit a signed transaction to Supra without double-spending or firing an irreversible write blind — estimate, simulate, submit, then confirm, using the account sequence number as the idempotency key.
api: Supra RPC Node API
base_url: https://rpc-mainnet.supra.com
operations:
  - chain_id
  - transaction_parameter
  - get_account_v3
  - estimate_gas_price_v3
  - simulate_txn_v3
  - submit_txn_v3
  - transaction_by_hash_v4
generated: '2026-08-29'
method: generated
source: >-
  openapi/supra-rpc-node-openapi.yml and conventions/supra-conventions.yml. Every operationId is grepped from
  the served contract; the irreversibility statement is the contract's own semantics, not an inference.
---

# Submit a Supra transaction safely

**Read this first: a committed Supra transaction cannot be cancelled, refunded, voided or
rolled back.** There is no reversal operation anywhere in the API. The compensating control
is to rehearse before you act, and the API gives you a real one.

## 1. Assert the network

    GET /rpc/v1/transactions/chain_id     # chain_id

`8` = mainnet, `6` = testnet. Fail closed if it is not the value your caller intended. This
is the cheapest guard against the single worst failure mode — signing a mainnet transaction
you meant for testnet.

## 2. Read the parameters you must sign within

    GET /rpc/v1/transactions/parameters   # transaction_parameter
    GET /rpc/v3/transactions/estimate_gas_price  # estimate_gas_price_v3

`transaction_parameter` returns the network's current bounds (max gas amount, expiry
bounds). `estimate_gas_price_v3` returns the current gas price. Sign inside these or the
node rejects the transaction with 400.

## 3. Take the sequence number — this is your idempotency key

    GET /rpc/v3/accounts/{address}        # get_account_v3

`AccountData.sequence_number` is the per-account monotonic counter. A signed transaction
commits to exactly one sequence number, and a consumed sequence number can never be reused.

**This is what makes retry safe.** If your submit call times out and you do not know whether
it landed, you may resubmit the *identical signed bytes* — the network cannot execute them
twice. Do not re-sign with a fresh sequence number to "retry"; that is how you double-spend.

There is no `Idempotency-Key` header on this API. If you are porting a Stripe-shaped client,
this is where the mapping differs.

## 4. Simulate

    POST /rpc/v3/transactions/simulate    # simulate_txn_v3

Body: the same `SupraTransaction` shape you are about to submit. Returns the gas used, the
`TxExecutionStatus`, and the write set the transaction would produce — without touching
consensus. Check `status` is success and the write set matches what you expect **before**
step 5. If you skip this, you have no dry run and no undo.

## 5. Submit

    POST /rpc/v3/transactions/submit      # submit_txn_v3

Content types accepted:

- `application/json` with a `SupraTransaction`
- `application/x.supra.signed_transaction+bcs` with the BCS-serialized signed bytes

Returns the transaction `Hash`, plus an `x-supra-chain-id` response header. **Assert that
header matches step 1** — it is the network telling you which chain actually accepted the
write.

Do not use `/rpc/v1/transactions/submit` (`submit_txn`); it is marked `deprecated: true`.

## 6. Confirm

    GET /rpc/v3/transactions/{hash}       # transaction_by_hash_v3
    GET /rpc/v4/transactions/{hash}       # transaction_by_hash_v4

Poll until the transaction appears with a `TransactionOutputV4` and a terminal
`TxExecutionStatus`. `404` here means "not yet", not "failed" — a submitted transaction is
not instantly queryable. Use backoff; do not resubmit while polling.

For a stronger guarantee, `transaction_by_hash_v4` carries an `inclusion_proof`
(`TransactionInclusionProof`) you can verify against the certified Merkle root rather than
trusting the node's answer.

## What you cannot do

| You might want | Reality |
|---|---|
| Cancel a submitted transaction | No such operation exists |
| Refund / reverse a transfer | Only by a second transaction the counterparty authorizes |
| An `Idempotency-Key` header | Not present — use the sequence number |
| A sunset date for the deprecated v1/v2 paths | Not published |

See `conventions/supra-conventions.yml` (reversibility) and `lifecycle/supra-lifecycle.yml`.
