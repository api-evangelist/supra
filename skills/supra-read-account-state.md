---
name: supra-read-account-state
description: Read everything the Supra Layer 1 knows about an account — sequence number, authentication key, published Move modules, stored Move resources, and transaction history — using the keyless public RPC node API.
api: Supra RPC Node API
base_url: https://rpc-mainnet.supra.com
operations:
  - get_account_v3
  - get_account_resources_v3
  - get_account_resource_v3
  - get_account_modules_v3
  - get_account_module_v3
  - get_account_transactions_v3
  - coin_transactions_v3
  - fa_transactions_v3
  - statement_automated_v3
generated: '2026-08-29'
method: generated
source: openapi/supra-rpc-node-openapi.yml — every operationId above is grepped from the served contract.
---

# Read Supra account state

No credential is required. The node API is public and keyless; do not send an Authorization
header or an API key.

## 1. Confirm which network you are on

    GET /rpc/v1/transactions/chain_id        # operationId: chain_id

Returns `8` on mainnet, `6` on testnet. Do this before you interpret any address —
the same address exists on both networks with different state.

## 2. Fetch the account

    GET /rpc/v3/accounts/{address}           # operationId: get_account_v3

`address` accepts a 256-bit address with or without the `0x` prefix. Returns `AccountData`:
`sequence_number` and `authentication_key`. That is all an account *is* on Supra — the
interesting state lives in the resources published under it.

Do not use `/rpc/v2/accounts/{address}` (`get_account_v2`). It is marked `deprecated: true`
in the contract.

## 3. List what is stored under it

    GET /rpc/v3/accounts/{address}/resources                 # get_account_resources_v3
    GET /rpc/v3/accounts/{address}/resources/{resource_type} # get_account_resource_v3
    GET /rpc/v3/accounts/{address}/modules                   # get_account_modules_v3
    GET /rpc/v3/accounts/{address}/modules/{module_name}     # get_account_module_v3

`resource_type` is a fully qualified Move struct tag, e.g.
`0x1::coin::CoinStore<0x1::supra_coin::SupraCoin>`. URL-encode it.
`get_account_module_v3` returns `MoveModuleBytecode`, which carries the module's `abi`
(a `MoveModule`) — that ABI is what tells you which functions are callable and which are
`view`.

## 4. Page the history

    GET /rpc/v3/accounts/{address}/transactions       # get_account_transactions_v3
    GET /rpc/v3/accounts/{address}/coin_transactions  # coin_transactions_v3
    GET /rpc/v3/accounts/{address}/fa_transactions    # fa_transactions_v3
    GET /rpc/v3/accounts/{address}/automated_transactions  # statement_automated_v3

Pagination, per the contract:

- `count` — maximum items, default 20.
- `start` — the cursor. On `get_account_transactions_v3` this is a **sequence number**.
  On the coin and fungible-asset statement endpoints it is a **derived cursor**
  (block timestamp plus the transaction's zero-based index within its block); pass back
  the `x-supra-cursor` header returned by the previous page.
- `ascending` — defaults to `false`.

The direction flips depending on `start`. Omit `start` and you get the most recent `count`
items in **descending** order; supply `start` and you get `count` items from that cursor in
**ascending** order. Getting this wrong silently pages the wrong way rather than erroring.

## 5. Handle pruning

Nodes prune. Six operations, including these statement endpoints, declare `410 Gone`:
"Requested data has been pruned and is no longer available."

**410 is permanent for this node — do not retry it.** Read the `x-supra-oldest-block`
response header to learn the retention floor, then narrow your range or query an archival
node. Retry logic that treats 410 as transient will loop forever.

## Errors

The envelope is `{"message": "<string>"}` — one string, no error code, no RFC 9457.
Branch on the HTTP status:

| Status | Meaning | Retry |
|---|---|---|
| 400 | Malformed address or hash | No — fix the input |
| 404 | No such account / height not yet produced | Only if the height is at the chain tip |
| 410 | Pruned | Never |
| 503 | Archive still initializing | Yes, with backoff |

See `errors/supra-problem-types.yml`.
